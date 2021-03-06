# DASCRUBBER wrapper

Gene Myers produced a nice set of tools for scrubbing long reads: trimming them, removing chimeras and patching up low quality regions. While raw long reads can contain a fair bit of junk, scrubbed reads should all be contiguous pieces of the underlying sequence, and this makes assembly much easier. Read all about it on his blog: [Scrubbing Reads for Better Assembly](https://dazzlerblog.wordpress.com/2017/04/22/1344/)

I had two issues with DASCRUBBER. First, it only works for PacBio reads, because PacBio FASTA headers are needed to build a Dazzler database. Second, it is not simple to run, involving more than 10 separate tools and commands. I wrote this wrapper script to solve these issues: it carries out the entire DASCRUBBER process with a single, easy to run command, and it works on any set of long reads (including Oxford Nanopore reads) by faking PacBio read names.

Disclaimer: this wrapper was designed for small genomes where the read sets aren't too large. If you have a large genome or read set, you may run into system resource problems. [Read more here.](#large-datasets)



## Table of contents

* [Requirements](#requirements)
* [Installation](#installation)
* [Example commands](#example-commands)
* [Method](#method)
* [Parameters](#parameters)
* [Full usage](#full-usage)
* [Large datasets](#large-datasets)
* [License](#license)



## Requirements

You'll need a number of Dazzler tools available in your PATH: [DAZZ_DB](https://github.com/thegenemyers/DAZZ_DB), [DALIGNER](https://github.com/thegenemyers/DALIGNER), [DAMASKER](https://github.com/thegenemyers/DAMASKER) and [DASCRUBBER](https://github.com/thegenemyers/DASCRUBBER)

This bash loop should clone and build them. Just replace `~/.local/bin` with wherever you want to put the executables:

```
for repo in DAZZ_DB DALIGNER DAMASKER DASCRUBBER; do
    git clone https://github.com/thegenemyers/"$repo"
    cd "$repo" && make -j && cd ..
    find "$repo" -maxdepth 1 -type f -executable -exec cp {} ~/.local/bin \;
done
```



## Installation

This tool is a single Python 3 script with no third-party dependencies. It will run without any installation:
```
git clone https://github.com/rrwick/DASCRUBBER-wrapper
DASCRUBBER-wrapper/dascrubber_wrapper.py --help
```

If you want, you can copy the it to somewhere in your PATH for easy usage:
```
cp DASCRUBBER-wrapper/dascrubber_wrapper.py ~/.local/bin
dascrubber_wrapper.py --help
```



## Example commands

This script has two required parameters: input reads and genome size. It outputs scrubbed reads to stdout, so its output should be directed to a file.

__Default parameters for a 5.5 Mbp bacterial genome:__<br>
`dascrubber_wrapper.py -i reads.fastq.gz -g 5.5M | gzip > scrubbed.fasta.gz`

__Limit daligner memory usage to 50 GB:__<br>
`dascrubber_wrapper.py -i reads.fastq.gz -g 5.5M --daligner_options="-M50" | gzip > scrubbed.fasta.gz`

__Keep Dazzler files after completion:__<br>
`dascrubber_wrapper.py -i reads.fastq.gz -g 5.5M -d working_files --keep | gzip > scrubbed.fasta.gz`



## Method

1. Convert reads to a FASTA file with PacBio-style headers.
    * It is assumed that the input reads are either FASTQ or FASTA (one line per sequence). Gzipped reads are okay. Multi-line-per-read FASTA files are not okay (I might get around to adding that later).
2. Build a [Dazzler database](https://dazzlerblog.wordpress.com/2016/05/21/dbs-and-dams-whats-the-difference/) of the reads with [fasta2DB](https://dazzlerblog.wordpress.com/command-guides/dazz_db-command-guide/).
3. Split the database with [DBsplit](https://dazzlerblog.wordpress.com/command-guides/dazz_db-command-guide/).
    * Splitting seems to be necessary or else the DASedit command will fail – I'm not sure why.
4. [Find all read-read overlaps](https://dazzlerblog.wordpress.com/2014/07/10/dalign-fast-and-sensitive-detection-of-all-pairwise-local-alignments/) with [daligner](https://dazzlerblog.wordpress.com/command-guides/daligner-command-reference-guide/).
    * This is the slowest and most memory-hungry step of the process.
5. [Mask repeats](https://dazzlerblog.wordpress.com/2016/04/01/detecting-and-soft-masking-repeats/) with [REPmask](https://dazzlerblog.wordpress.com/command-guides/damasker-commands/).
    * See the [Parameters](#parameters) section for details on the repeat depth threshold.
6. Find tandem repeats with [datander](https://dazzlerblog.wordpress.com/command-guides/damasker-commands/).
7. Mask tandem repeats with [TANmask](https://dazzlerblog.wordpress.com/command-guides/damasker-commands/).
8. Find all read-read overlaps with [daligner](https://dazzlerblog.wordpress.com/command-guides/daligner-command-reference-guide/) again, this time masking repeats.
9. Find estimated genome coverage with [DAScover](https://dazzlerblog.wordpress.com/command-guides/dascrubber-command-guide/).
10. Find [intrinsic quality values](https://dazzlerblog.wordpress.com/2015/11/06/intrinsic-quality-values/) with [DASqv](https://dazzlerblog.wordpress.com/command-guides/dascrubber-command-guide/).
11. [Trim reads and split chimeras](https://dazzlerblog.wordpress.com/2017/04/22/1344/) with [DAStrim](https://dazzlerblog.wordpress.com/command-guides/dascrubber-command-guide/).
    * See the [Parameters](#parameters) section for details on the good and bad quality thresholds.
12. Patch low-quality regions of reads with [DASpatch](https://dazzlerblog.wordpress.com/command-guides/dascrubber-command-guide/).
13. Produce a new database of scrubbed reads with [DASedit](https://dazzlerblog.wordpress.com/command-guides/dascrubber-command-guide/).
14. Extract a FASTA of scrubbed reads with [DB2fasta](https://dazzlerblog.wordpress.com/command-guides/dazz_db-command-guide/).
15. Restore original read names and output to stdout.
    * A range is appended to the end of the new read names. For example, if the original read was named `read1975`, the scrubbed read might be named `read1975/400_9198`.
    * It is possible for one original read to result in more than one scrubbed read. For example, a chimeric read named `read2392` might result in two scrubbed reads: `read2392/0_12600` and `read2392/12700_25300`.



## Parameters

Parameters can be passed to any of the subcommands using the 'options' arguments. E.g. `--daligner_options`. Read Gene Myers' [command guides](https://dazzlerblog.wordpress.com/command-guides/) for details about the tools.

daligner can use a lot of memory – by default it will use all available memory on the system! I therefore find it useful to limit its memory usage, e.g. `--daligner_options="-M50"`. 

The REPmask command replies on a threshold depth, above which a sequence is considered to be a repeat (read more [here](https://dazzlerblog.wordpress.com/2016/04/01/detecting-and-soft-masking-repeats/)). I found that 3x the base depth works well for my datasets (high depth bacterial genomes). E.g. if the base depth (as determined using the genome size) is 50x then regions with 150x or greater depth are considered repeats. You can adjust this ratio from the default of 3 using the `--repeat_depth` option – a higher value may be appropriate for lower coverage datasets or highly repetitive genomes. Alternatively, you can manually set the threshold: e.g. `--repmask_options="-c40"`.

DAStrim takes two important parameters: `-g` and `-b` for good and bad quality thresholds. Its default behaviour is to use the 80th and 93rd percentiles of the QV scores made by DASqv. If you want to use different values, set them manually: e.g. `--dastrim_options="-g20 -b30"`.



## Full usage

```
usage: DASCRUBBER_wrapper.py -i INPUT_READS -g GENOME_SIZE [-d TEMPDIR] [-k]
                             [-r REPEAT_DEPTH]
                             [--dbsplit_options DBSPLIT_OPTIONS]
                             [--daligner_options DALIGNER_OPTIONS]
                             [--repmask_options REPMASK_OPTIONS]
                             [--datander_options DATANDER_OPTIONS]
                             [--tanmask_options TANMASK_OPTIONS]
                             [--dasqv_options DASQV_OPTIONS]
                             [--dastrim_options DASTRIM_OPTIONS]
                             [--daspatch_options DASPATCH_OPTIONS]
                             [--dasedit_options DASEDIT_OPTIONS] [-h]

A wrapper tool for the DASCRUBBER pipeline for scrubbing (trimming and chimera
removal) of long read sets (PacBio or ONT reads)

Required arguments:
  -i INPUT_READS, --input_reads INPUT_READS
                        input set of long reads to be scrubbed
  -g GENOME_SIZE, --genome_size GENOME_SIZE
                        approximate genome size (examples: 3G, 5.5M or 800k),
                        used to determine depth of coverage

Optional arguments:
  -d TEMPDIR, --tempdir TEMPDIR
                        path of directory for temporary files (default: use a
                        directory in the current location named
                        dascrubber_temp_PID where PID is the process ID)
  -k, --keep            keep the temporary directory (default: delete the
                        temporary directory after scrubbing is complete)
  -r REPEAT_DEPTH, --repeat_depth REPEAT_DEPTH
                        REPmask will be given a repeat threshold of this
                        depth, relative to the overall depth (e.g. if 3, then
                        regions with 3x the base depth are considered
                        repeats) (default: 3)

Command options:
  You can specify additional options for each of the Dazzler commands if you
  do not want to use the defaults (example: --daligner_options="-M80")

  --dbsplit_options DBSPLIT_OPTIONS
  --daligner_options DALIGNER_OPTIONS
  --repmask_options REPMASK_OPTIONS
  --datander_options DATANDER_OPTIONS
  --tanmask_options TANMASK_OPTIONS
  --dascover_options DASCOVER_OPTIONS
  --dasqv_options DASQV_OPTIONS
  --dastrim_options DASTRIM_OPTIONS
  --daspatch_options DASPATCH_OPTIONS
  --dasedit_options DASEDIT_OPTIONS

Help:
  -h, --help            show this help message and exit
```



## Large datasets

This wrapper isn't really suited to large datasets, because it does a simple all-vs-all read alignment with DALIGNER. For a very large read set, this will be too slow or use too much memory, and you'll instead want to use some of DALIGNER's HPC functionality. [Read more here.](https://dazzlerblog.wordpress.com/command-guides/daligner-command-reference-guide/)

The only benefit this wrapper might still have is its first step: faking PacBio read names, enabling the Dazzler tools to work for Nanopore reads. If you are in this position (wanting to scrub a large Nanopore dataset), then here's what I recommend:

1. Start this wrapper: `dascrubber_wrapper.py -i reads.fastq.gz -g 100M --tempdir renamed_reads` (it doesn't matter what you use for the genome size option)
2. When the wrapper finishes the 'Processing and renaming reads' step, kill it with Ctrl-C.
3. You should now have a directory named `renamed_reads` with a file inside named `renamed_reads.fasta`. You can delete anything else in that directory like the `reads.db` file or the `align_temp` directory, if they exist.
4. Take the renamed reads and run the database/aligning/scrubbing commands yourself. Plenty of information here: [dazzlerblog.wordpress.com](https://dazzlerblog.wordpress.com/)

Good luck!



## License

[GNU General Public License, version 3](https://www.gnu.org/licenses/gpl-3.0.html)

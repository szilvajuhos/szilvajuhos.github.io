---
title: 'Useful bash tricks '
date: 2020-11-08
permalink: /posts/2020/11/Useful_bash/
tags:
  - bash
  - linux
---
- [To get familiar with Sarek](#To-get-familiar-with-Sarek)
    - [Sarek use cases](#Sarek-use-cases)
    - [Running NA12878 germline](#Running-NA12878-germline)
    - [Run basic callers](#Run-basic-callers)
- [Reproduce things with conda](#Reproduce-things-with-conda)

### What is a pipeline?
When making a pipeline, we are hoping that it will:
- makes processing steps in a well defined order, consecutive steps creating dependencies
- when one of the steps fails, we are getting an error message. Furthermore, after fixing the error (i.e. adding 
a missing reference file), the pipeline will resume from the point of error
- there is a trace of logs about what hapened
- when running the same pipeline twice with the same data, we are getting the same results
- the pipeline hides processing details, number of CPUs, memory, whether it is a cluster or a single node, etc.

This way a shell script is not a pipeline. Nextflow itself is a programming language made to build pipelines, but 
it has a learning curve. On the other hand, many times when the pipeline encounters an error, it is not necessarily 
in the pipeline itself, but in the software that serves as a step in the pipeline, and hard to find what is the 
actual problem.

#### To get familiar with Sarek

The very first thing we want to try out whether it is working at all. There is a built-in test suite, that we can run:

```
$ mkdir /data1/junk
$ cd /data1/junk
$ nextflow run nf-core/sarek -r 2.6.1 -profile test,munin
N E X T F L O W  ~  version 20.07.1
Launching `nf-core/sarek` [festering_tuckerman] - revision: bce378e09d [2.6.1]
[ ... ]
Pipeline Release  : 2.6.1
Run Name          : festering_tuckerman
Max Resources     : 6 GB memory, 2 cpus, 2d time per job
Container         : singularity - nfcore/sarek:2.6.1
Input             : https://raw.githubusercontent.com/nf-core/test-datasets/sarek/testdata/tsv/tiny-manta-https.tsv
Step              : mapping
Genome            : smallGRCh37
Nucleotides/s     : 1000.0
MarkDuplicates    : Options
Java options      : "-Xms4000m -Xmx7g"
GATK Spark        : Yes
[...]
```

What is happening by launching this command? Nextflow is 
- running a pipeline (hence the `run` part - nextflow can do other things as well, not only running a pipeline)
- `nf-core/sarek` is the actual pipeline to launch. Using this sort of notation nextflow check github, looks for 
the `nf-core/sarek` repository, downloads it and runs
- `-r 2.6.1` refers to a particular version of Sarek. One can find out what tools are included in which version have
to read the CHANGELOG page: https://github.com/nf-core/sarek/blob/master/CHANGELOG.md
- `-profile test,munin` means that we want to run the tests on munin. When running things on munin, use the 
`-profile munin` directive only. Besides this, there are quite a few configs available, including UPPMAX and AWS 
(https://github.com/nf-core/configs) . To have a look at the test configuration of Sarek, check out https://github.com/nf-core/sarek/blob/master/conf/test.config  
The test should complete with a "Pipeline completed successfully" message. 

#### Sarek use cases:
There is already a collection of use cases in the documentation: https://github.com/nf-core/sarek/blob/master/docs/use_cases.md .
There are three things to add: the input, the step and the tools you want to use in a particular step.  

_Input_ can be defined either by making a TSV file, or, if starting from raw data, supplying a directory containing 
FASTQ files.

_Step_ is to choose from major processes, you can choose only one of them. Step can be "mapping", 
"prepare_recalibration", "recalibrate", "variant_calling", "annotate", "Control-FREEC". Control-FREEC has its own step
since preparing data for Control-FREEC is different. With _step_ we specify the starting point: all following steps 
will be run if they are available (dependencies met, for example recalibrated bams are prepared) and tools are defined
in the command line. I.e. variant calling will only be run if an available variant caller is specified 
with `--tools` directive, annotation will only be run if both a variant caller and an annotator is specified. This
command below will start mapping, once recalibrated BAMs are available starts Strelka, finally annotates the results 
with VEP: 
```
$ nextflow run nf-core/sarek -r 2.6.1 -profile munin --input sample.tsv --tools strelka,vep  
```

_Tools_ are the different software that we can call in a particular step. I.e.

There is a short description on how to make TSV files: https://github.com/nf-core/sarek/blob/master/docs/input.md . 
Below I consider an input file with a single sample, one tumour and one blood sample like:

```
G15511	XX	0	NORMAL_C09DFN   /path/to/C09DFN.bam	/path/to/C09DFN.bai
G15511	XX	1	TUMOR_D0ENMT    /path/to/D0ENMT.bam	/path/to/D0ENMT.bai
```  

When starting from raw FASTQ, Sarek expects all input files to be GZ compressed. 

#### Running NA12878 germline 
The well-knows NA12878 test set from the _Genome in a Bottle (GiaB)_ project is at 
/data2/NA12878/fastq/ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/NA12878/NIST_NA12878_HG001_HiSeq_300x/ on munin. To run a 
germline mapping one can start (note I am using `--sentieon` on munin to speed up things):
```
$ cd /data2/tutorial 
$ nextflow run nf-core/sarek -r 2.6.1 -profile munin --input \
na12878.tsv --sentieon 2>&1 | tee na12878.log
```

There is a funny [redirection](#run-a-script-and-log-its-output) at the end of the command (the `2>&1` part) to save
the output. Nevertheless, there is a `.nextflow.log` file always in the current directory to save the steps of the very
last run. One reason that you can not run multiple nextflow runs in the same directory is that the processes would
overwrite their logfiles. As the workflow progresses, you can check what is happening either by looking at the logfiles,
the terminal output or the other pipeline info files (details below about these). If you list all the `.nextflow.log*` 
files, you will see that there are actually many if you run nextflow several times before:   
```
(base) szilva@munin /data2/tutorial $ ls -l .nextflow.log*
-rw-rw-r-- 1 szilva   btb 371184 Nov 17 16:15 .nextflow.log
-rw-rw-r-- 1 szilva   btb  61389 Nov 17 09:32 .nextflow.log.1
-rw-rw-r-- 1 teresita btb 145069 Nov 16 14:11 .nextflow.log.2
-rw-rw-r-- 1 teresita btb   1010 Nov 13 09:46 .nextflow.log.3
-rw-rw-r-- 1 teresita btb 650798 Nov 12 17:09 .nextflow.log.4
-rw-rw-r-- 1 teresita btb  56373 Nov 12 09:38 .nextflow.log.5
-rw-rw-r-- 1 teresita btb 156757 Nov 12 09:29 .nextflow.log.6
-rw-rw-r-- 1 szilva   btb 154204 Nov 12 09:26 .nextflow.log.7
-rw-rw-r-- 1 teresita btb 143895 Nov 12 09:26 .nextflow.log.8
-rw-rw-r-- 1 teresita btb   4265 Nov 12 09:24 .nextflow.log.9
```

This means every time we launch a new run, the old logfile will be renamed as `.nextflow.log.1`. If there is already
a logfile called `.nextflow.log.1`, that will be renamed as `.nextflow.log.2` and so on. This is also true to the
pipeline info file one can find in the `results/pipeline_info` directory:
```
(base) szilva@munin /data2/tutorial $ ls -l results/pipeline_info/
total 33216
-rw-rw-r-- 1 szilva   btb 2936288 Nov 17 09:32 execution_report.html
-rw-rw-r-- 1 teresita btb 3055459 Nov 16 14:11 execution_report.html.1
[...]
-rw-rw-r-- 1 szilva   btb 3056607 Nov 11 17:42 execution_report.html.9
-rw-rw-r-- 1 szilva   btb    7124 Nov 17 09:32 execution_timeline.html
-rw-rw-r-- 1 teresita btb   30397 Nov 16 14:11 execution_timeline.html.1
[...]
-rw-rw-r-- 1 szilva   btb   30436 Nov 11 17:42 execution_timeline.html.9
-rw-rw-r-- 1 szilva   btb   47582 Nov 17 15:42 execution_trace.txt
-rw-rw-r-- 1 szilva   btb    1065 Nov 17 09:32 execution_trace.txt.1
[...]
-rw-rw-r-- 1 szilva   btb     101 Nov 12 09:23 execution_trace.txt.9
-rw-rw-r-- 1 szilva   btb  279740 Nov 17 09:32 pipeline_dag.svg.1
[...]
-rw-rw-r-- 1 szilva   btb  284442 Nov 11 17:42 pipeline_dag.svg.9
-rw-rw-r-- 1 szilva   btb   21036 Nov 17 09:32 pipeline_report.html
-rw-rw-r-- 1 szilva   btb    3940 Nov 17 09:32 pipeline_report.txt
-rw-r--r-- 1 szilva   btb   44152 Nov 17 09:40 results_description.html
-rw-r--r-- 1 szilva   btb     373 Nov 17 09:35 software_versions.csv
```

These files are:
 - execution_report.html : gives a pretty output about tasks, CPU and IO usage 
 - execution_timeline.html : timing summary of different tasks
 - ***execution_trace.txt*** : this is what I like the most: runtime and IO statistics of completed tasks in plain 
 text. Can be processed with awk or loaded into Excel. 
 - pipeline_dag.svg : [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) representation of the workflow 
 that is more impressive than useful
 - pipeline_report.html : launching parameters in html
 - pipeline_report.txt : same information in text
 - results_description.html : description of used tools in html
 - software_versions.csv : version of each tool used

When running a task like the one above, mapping will be done separately for each line in the TSV file processing 
different reads groups in separate tasks. At the end there will be a merging step, afterwards deduplication and 
recalibration. It is important to know that the deduplicated BAM file still contains all the reads, but the recalibrated
one _does not_: centromeres and telomeres, long tandem repeat regions are missing. 

#### Run basic callers

The two core germline callers are HaplotypeCaller and Strelka, furthermore we can have Manta for structural variants.
It is advised to run Strelka and Manta together, as the default behaviour of Sarek is to run the Strelka best practices 
pipeline where Manta creates a candidate list for small indels. Having recalibrated bams to call them is like:

```
nextflow run nf-core/sarek -r 2.6.1 -profile munin --step variantcalling --input test.tsv --tools haplotypecaller,strelka
```

Note, the `nextflow run nf-core/sarek -r 2.6.1 -profile munin` part is and will be just the same in all cases. If 
you are creating a shell script to run things, better to write something like:

```
#/bin/bash -ex
NFXRUN="nextflow run nf-core/sarek -r 2.6.1 -profile munin"
${NFXRUN} --step ....(rest of the command)
```  


### Reproduce things with conda
Many times you want to install some software, but you not want to bother the system manager. 
Or, your belowed software needs version 3.4.5 of some other software, but you have 1.2.3 installed. 
Conda (notably [Bioconda](https://bioconda.github.io/user/install.html) ) can resolve this for you by installing
software to your local directories and by using environments that are fine-tuned for compatible versions. There are 
many different people using conda, not only bioinformaticians. The core conda packages are those that are used by 
practically everybody, but to add biology-focused packages, it is advised to add 
[other channels](https://bioconda.github.io/user/install.html#set-up-channels) like:

```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
```

After installing and adding channels to conda next step is to use _environments_. Without these we can not install
multiple versions of different software. To activate the base environment (that contains only base packages) use:
```
$ conda activate base
(base) $   
```
All new environment should be based on _base_, but we should not bloat base with all the software we are using.
Instead, make new environments and delete unused ones. An example of creating two environments that contain
different versions of GATK:
```
(base) $ conda search gatk
Loading channels: done
# Name                       Version           Build  Channel             
gatk                             3.5              10  bioconda            
...
gatk                             3.7          py36_1  bioconda            
gatk                             3.8              10  bioconda            
...
```
(it will list quite a few other versions as well). In fact, `conda search gatk4` will list even more entries as the 
Broad team started to use conda heavily. Now we will make two GATK environments: one for GATK 3.8, the other for GATK4. 
First step is to create new environments:
```
$ conda activate base   # important to be in the base environment 
(base) $ conda create -n gatk38
Collecting package metadata (current_repodata.json): done
Solving environment: done
## Package Plan ##
  environment location: /home/szilva/miniconda3/envs/gatk38
Proceed ([y]/n)?
[...]
```   
Creating a new environment will not activate it yet, so we can make the gatk4 version as well by typing 
`conda create -n gatk4` . If we are lost in different conda commands (as I am all the time), just type `conda` with 
no arguments, it will give you a well structured help system. Once we have the two GATK environments ready, time to 
activate one of them, and add the appropriate GATK to it:
```
$ conda activate gatk38
(base) $ conda activate gatk38
(gatk38) $ conda install gatk
Collecting package metadata (current_repodata.json): done
Solving environment: done

## Package Plan ##
  environment location: /home/szilva/miniconda3/envs/gatk38
  added / updated specs:
    - gatk

The following packages will be downloaded:
    package                    |            build
    ---------------------------|-----------------
    binutils_impl_linux-64-2.35.1|       h17ad2fc_0         9.3 MB  conda-forge
    bwidget-1.9.14             |       ha770c72_0         119 KB  conda-forge
    ca-certificates-2020.11.8  |       ha878542_0         145 KB  conda-forge
...
```
If we are not adding version, conda will install the latest version of a particular package. To install GATK 3.X one 
have to have many different packages, with particular versions, but conda will handle this for us. We can check whether 
GATK is installed:
```
(gatk38) $ gatk3
##### ERROR ------------------------------------------------------------------------------------------
##### ERROR A USER ERROR has occurred (version 3.8-1-0-gf15c1c3ef): 
##### ERROR
##### ERROR This means that one or more arguments or inputs in your command are incorrect.
##### ERROR The error message below tells you what is the problem.
[...]
##### ERROR MESSAGE: Argument with name '--analysis_type' (-T) is missing.
##### ERROR ------------------------------------------------------------------------------------------
``` 
We can go through the same with GATK4, first activating the still empty GATK4 environment, and 
installing the latest version:

```
(gatk38) $ conda activate gatk4
(gatk4) $ conda install gatk4
Collecting package metadata (current_repodata.json): done
Solving environment: done

## Package Plan ##
  environment location: /home/szilva/miniconda3/envs/gatk4
  added / updated specs:
    - gatk4

The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    gatk4-4.1.9.0              |           py39_0       276.5 MB  bioconda
    ------------------------------------------------------------
                                           Total:       276.5 MB
[...]

(gatk4) $ gatk 

 Usage template for all tools (uses --spark-runner LOCAL when used with a Spark tool)
    gatk AnyTool toolArgs

 Usage template for Spark tools (will NOT work on non-Spark tools)
    gatk SparkTool toolArgs  [ -- --spark-runner <LOCAL | SPARK | GCS> sparkArgs ]

 Getting help
    gatk --list       Print the list of available tools

    gatk Tool --help  Print help on a particular tool
[...]
```

Once we are done with environments, we can deactivate them by `conda deactivate`. Environments are hierarchical, 
when we are creating one, the actual enviroment will be considered as "base", meaning new software will be installed 
on top of the current packages. Hence, if we are installing many software into `base`, quickly there will be a 
version conflict. On the other hand, if we are using different environment for different tasks, we will use up disk 
space, but that is negligible compared to NGS data. 

#### Deleting a broken environment
If we screw up an environment, the best is to get rid of it; first I list the available environments, than delete
the environment called `bugger`:
```
(base) $ conda env list
# conda environments:
#
base                  *  /home/szilva/miniconda38
bugger                   /home/szilva/miniconda38/envs/bugger
gatk38                   /home/szilva/miniconda38/envs/gatk38
gatk4                    /home/szilva/miniconda38/envs/gatk4

(base) $ conda env remove -n bugger

Remove all packages in environment /home/szilva/miniconda38/envs/bugger:

(base) $ conda env list
# conda environments:
#
base                  *  /home/szilva/miniconda38
gatk38                   /home/szilva/miniconda38/envs/gatk38
gatk4                    /home/szilva/miniconda38/envs/gatk4
```      

#### Installing software with a given version
I am using a test environment I have created to check various versions of samtools, and will install an earlier
version, i.e. version 1.3.1:
```
(test) $ samtools
bash: samtools: command not found
(test) $ conda search samtools
Loading channels: done
# Name                       Version           Build  Channel
samtools                      0.1.12               0  bioconda
samtools                      0.1.12               1  bioconda
samtools                      0.1.12               2  bioconda
samtools                      0.1.13               0  bioconda
[ ... this actually takes some time ... ]
(test) $ conda install samtools=1.3.1
Collecting package metadata (current_repodata.json): done
Solving environment: done
## Package Plan ##
  environment location: /home/szilva/miniconda38/envs/test
  added / updated specs:
    - samtools=1.3.1

The following packages will be downloaded:
    package                    |            build
    ---------------------------|-----------------
    c-ares-1.16.1              |       h516909a_3         107 KB  conda-forge
    curl-7.71.1                |       he644dc0_8         139 KB  conda-forge
    krb5-1.17.1                |       hfafb76e_3         1.5 MB  conda-forge
[...]
Proceed ([y]/n)?

Downloading and Extracting Packages
krb5-1.17.1          | 1.5 MB    | #########
[...]
(test) $ samtools

Program: samtools (Tools for alignments in the SAM format)
Version: 1.3.1 (using htslib 1.3.1)

Usage:   samtools <command> [options]
[...]
``` 

#### Updating to the latest version
Simple updating _should_ work as `conda update samtools`. Life is hard, i.e. it is not working as expected, my guess is 
because the difference between samtools 1.3.1 and 1.11 (the latest) is too large:
```
(test) $ conda update samtools
Collecting package metadata (current_repodata.json): done
Solving environment: done
## Package Plan ##
  environment location: /home/szilva/miniconda38/envs/test
  added / updated specs:
    - samtools
The following NEW packages will be INSTALLED:
  bzip2              conda-forge/linux-64::bzip2-1.0.8-h516909a_3
  libgcc             conda-forge/linux-64::libgcc-7.2.0-h69d50b8_2
  xz                 conda-forge/linux-64::xz-5.2.5-h516909a_1
The following packages will be UPDATED:
  samtools                                 1.3.1-h80b0bb3_7 --> 1.7-1
Proceed ([y]/n)? y
[...]
Executing transaction: done
(test) $ samtools
samtools: error while loading shared libraries: libcrypto.so.1.0.0: cannot open shared object file: No such file or directory
```
We can see it was not even updating to the latest version (1.11). Whatever, we can either forcing reinstall, and hope the 
best, or delete this broken environment, create a new one, and run `conda install samtools` that should do the latest
version. Forcing reinstall works in this case:
```
(test) $ conda install samtools=1.11
Collecting package metadata (current_repodata.json): done
Solving environment: failed with initial frozen solve. Retrying with flexible solve.
Solving environment: failed with repodata from current_repodata.json, will retry with next repodata source.
Collecting package metadata (repodata.json): done  
Solving environment: done
[...]
(test) $ samtools --version
samtools 1.11
Using htslib 1.11
Copyright (C) 2020 Genome Research Ltd.
```
The general rule is that do not expect all the version jumps are working (i.e. we already know that GATK 3.X to 
GATK 4.X transition is hopeless), but `conda update package-name` should work in most cases. 


### reproduce things with singularity
singularity shell

### Where is my file
You have 200T data around and you are remembering only vaguely the name of the file

### Is it the same
Compare files
- md5sum
- diff, sdiff

### How many of them
- wc -l
- grep -c

### if you are still interested in modules ...
will write if have time

### yum is your friend

### run a script and log its output

./runBugger.sh 2>&1 | tee soffer.`date +%Y-%m-%d-%Hh%Mm`.log

### What chromosomes are in this bloody large FASTA?
```awk '/>/{print}' /data1/references/igenomes/Homo_sapiens/GATK/GRCh38/Sequence/WholeGenomeFasta/Homo_sapiens_assembly38.fasta```

### htop stopping nexftlow and all of my runaway stuff

### too many arguments, xargs magic

### basename readlink

### what env do I have?

### if not vim, use nano






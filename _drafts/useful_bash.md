---
title: 'Useful bash tricks '
date: 2020-11-08
permalink: /posts/2020/11/Useful_bash/
tags:
  - bash
  - linux
---
## Useful bash and other tricks
 
- [Pipelines in general](#pipelines-in-general-)
- [To get familiar with Sarek](#to-get-familiar-with-sarek-)
    - [Sarek use cases](#Sarek-use-cases-)
    - [Running NA12878 germline](#Running-NA12878-germline-)
    - [Run basic callers](#Run-basic-callers-)
- [Reproduce things with conda](#Reproduce-things-with-conda-)
    - [Deleting a broken environment](#deleting-a-broken-environment-)
    - [Installing software with a given version](#installing-software-with-a-given-version-)
    - [Updating to the latest version](#updating-to-the-latest-version-)
- [Reproduce things with singularity](#Reproduce-things-with-singularity-)
- [Where is my file?](#Where-is-my-file-)
- [Is it the same?](#is-it-the-same-)
- [All I want is fusion](#all-i-want-is-fusion-)
- [BAM subset](#BAM-subset-)
- [How many of them?](#How-many-of-them-)
- [Run a script and log its output](#run-a-script-and-log-its-output-)
- [What chromosomes are in this bloody large FASTA?](#what-chromosomes-are-in-this-bloody-large-fasta-)
- [htop, stopping nexftlow and all of my runaway stuff](#htop-stopping-nextflow-and-all-of-my-runaway-stuff-)
- [Too many arguments, xargs magic](#too-many-arguments-xargs-magic-)
- [Get the filename only, or full path](#get-the-filename-only-or-full-path-)
- [What env do I have?](#What-env-do-I-have-)

### Pipelines in general [^](#Useful-bash-and-other-tricks)
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

#### To get familiar with Sarek [^](#Useful-bash-and-other-tricks)

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

#### Sarek use cases: [^](#Useful-bash-and-other-tricks)
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

#### Running NA12878 germline [^](#Useful-bash-and-other-tricks)
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

#### Run basic callers [^](#Useful-bash-and-other-tricks)
If we are running Sarek with only the `--input` option, it will be the mapping step that will be finished,
no variant callers are called since no tools were defined. If we add a variant caller (alternatively an annotation tool),
it will proceed further, and after mapping variant calling and annotation will be executed. On the other hand, if
we already have pre-processed BAM files, we have to define the step `--varinatcalling`, otherwise the BAM files are
going to be interpreted as BAMs waiting to be re-mapped, as the default first step is `--mapping` .
The two core germline callers are HaplotypeCaller and Strelka, furthermore we can have Manta for structural variants.
It is advised to run Strelka and Manta together, as the default behaviour of Sarek is to run the Strelka best practices 
pipeline where Manta creates a candidate list for small indels. Having recalibrated bams to call them is like:

```
nextflow run nf-core/sarek -r 2.6.1 -profile munin --step variantcalling \
    --input test.tsv --tools haplotypecaller,strelka
```

Note, the `nextflow run nf-core/sarek -r 2.6.1 -profile munin` part is and will be just the same in all cases. If 
you are creating a shell script to run things, better to write something like:

```
#!/bin/bash -xe
NFXRUN="nextflow run nf-core/sarek -r 2.6.1 -profile munin"
${NFXRUN} --step ....(rest of the command)
```  

About the `-xe` flag (and other optional flags to make bash safer to run) have a look at 
[this](https://vaneyckt.io/posts/safer_bash_scripts_with_set_euxo_pipefail/) post.


### Reproduce things with conda [^](#Useful-bash-and-other-tricks)

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
when we are creating one, the actual environment will be considered as "base", meaning new software will be installed 
on top of the current packages. Hence, if we are installing many software into `base`, quickly there will be a 
version conflict. On the other hand, if we are using different environment for different tasks, we will use up disk 
space, but that is negligible compared to NGS data. 

#### Deleting a broken environment [^](#Useful-bash-and-other-tricks)

If we screw up an environment, the best is to get rid of it; first I list the available environments, then delete
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

#### Installing software with a given version [^](#Useful-bash-and-other-tricks)

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

#### Updating to the latest version [^](#Useful-bash-and-other-tricks)

Simple updating _should_ work as `conda update samtools`. Life is hard, for samtools it is not working as expected, 
my guess because the difference between samtools 1.3.1 and 1.11 (the latest) is too large:
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
samtools: error while loading shared libraries: libcrypto.so.1.0.0: cannot open shared object file: No such file or directory.
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


### Reproduce things with singularity [^](#Useful-bash-and-other-tricks)

The extreme version of using restricted, well-defined environments is using [singularity](https://sylabs.io/docs/) 
containers. Singularity is practically a small machine inside a machine. On our server containers are at 
`/data1/containers`, the most straightforward way to start the Sarek container is like:
```
$ singularity shell /data1/containers/nfcore-sarek-2.6.1.img
Singularity nfcore-sarek-2.6.1.img:~>
```
It will have a new prompt, has little to do with our _private_ conda, and going to include only software that are 
included in the container. There is a conda environment though that is valid for this container, you can check its 
location with `conda env list`. It is pointing to `/opt/conda/envs/nf-core-sarek-2.6.1`, where the Sarek binaries are
installed. Since Sarek contains BWA, Control-FREEC, Manta, etc, you can find these pre-installed in the 
`/opt/conda/envs/nf-core-sarek-2.6.1/bin` directory, and can try out them:

```
Singularity nfcore-sarek-2.6.1.img:~> freec
Control-FREEC v11.5 : a method for automatic detection of copy number alterations, subclones and 
for accurate estimation of contamination and main ploidy using deep-sequencing data
	Please specify a config file
[...]
Singularity nfcore-sarek-2.6.1.img:~> configManta.py 
Usage: configManta.py [options]

Version: 1.6.0
[...]
Singularity nfcore-sarek-2.6.1.img:~> gatk Mutect2
Using GATK jar /opt/conda/envs/nf-core-sarek-2.6.1/share/gatk4-4.1.7.0-0/gatk-package-4.1.7.0-local.jar
Running:
    java -Dsamjdk.use_async_io_read_samtools=false -Dsamjdk.use_async_io_write_samtools=true -Dsamjdk.use_async_io_write_tribble=false -Dsamjdk.compression_level=2 -jar /opt/conda/envs/nf-core-sarek-2.6.1/share/gatk4-4.1.7.0-0/gatk-package-4.1.7.0-local.jar Mutect2
USAGE: Mutect2 [arguments]
Call somatic SNVs and indels via local assembly of haplotypes
Version:4.1.7.0
Required Arguments:
--input,-I:String             BAM/SAM/CRAM file containing reads  This argument must be specified at least once.
                              Required.
[...]
```

This way you can have software installed without pestering the sysadmin _and_ versioning is also controlled: one
container will use one well defined version of the software. In fact you do not have to go into the singularity shell, 
you can use `exec` to run a program included in the container:
  
```
$ singularity exec /data1/containers/nfcore-sarek-2.6.1.img freec -conf bugger.conf
Control-FREEC v11.5 : a method for automatic detection of copy number alterations, 
subclones and for accurate estimation of contamination and main ploidy using deep-sequencing data
        Could not find your config file.. Please, check the existance of bugger.conf
```

### Where is my file? [^](#Useful-bash-and-other-tricks)
You have 200T data around and you are remembering only vaguely the name of the file. Munin has a (quite standard) 
database that is updated every evening, and can be searched with the command `locate`. (Since this database is updated
every evening by scanning through the disks, it is an other reason to clean up stuff.) If trying to find a pattern like 
`Homo_sapiens`, only type: 

```
$ locate Homo_sapiens| grep -v work |head
/data0/btb/VarSeq/Common Data/Annotations/ClinVar2019-08-01-NCBI_GRCh_38_Homo_sapiens.tsf
/data0/btb/VarSeq/Common Data/Annotations/ClinVarTranscriptCounts2019-08-01-NCBI_GRCh_38_Homo_sapiens.tsf
/data0/btb/VarSeq/Common Data/Annotations/ConservationScoresExonic-GHI_2018-05-01_GRCh_38_Homo_sapiens.tsf
[...]
``` 

You can use [regexp](https://en.wikipedia.org/wiki/Regular_expression), or you can pipe it into grep, awk whatever:
 
```
$ locate *UCSC_GRCh_[0-9][0-9]_*_Homo_sapiens*| egrep -v "tere|old|softw"
/data0/btb/VarSeq/Common Data/Annotations/Cytobands2009-06-12-UCSC_GRCh_37_g1k_Homo_sapiens.tsf
/data1/VarSeq/Common Data/Annotations/Cytobands2009-06-12-UCSC_GRCh_37_g1k_Homo_sapiens.tsf
```

Though if you create a file, changes in the `locate` database will be updated only next evening, new files
are not visible immediately (`touch` creates a new empty file):

```
$ touch mynewfilewithalongname
$ locate mynewfilewithalongname
$
```

An other way to find files is use the command `find` . It is a bit more complicated to use, but has an immediate effect:
```
$ find . -name "*newfilewith*"
./mynewfilewithalongname
``` 

The dot (`.`) in this command means "current directory", the first argument of `find` is always the dir where we want to 
start the search. I.e. searching for PDF files in the `/data2/junk` directory is like:
```
$ find /data2/junk -name "*.pdf"
/data2/junk/results/Reports/MultiQC/multiqc_plots/pdf/mqc_fastqc_sequence_counts_plot_1.pdf
/data2/junk/results/Reports/MultiQC/multiqc_plots/pdf/mqc_fastqc_sequence_counts_plot_1_pc.pdf
/data2/junk/results/Reports/MultiQC/multiqc_plots/pdf/mqc_fastqc_per_base_sequence_quality_plot_1.pdf
/data2/junk/results/Reports/MultiQC/multiqc_plots/pdf/mqc_fastqc_per_sequence_quality_scores_plot_1.pdf
[...]
```
Can search for directories only with the `-type d` directive (searching for files is `-type f`):

```
$ find /data2/junk -type d -name "pipeline*"
/data2/junk/results/pipeline_info
```

And we can chain logical operators like `-and -not` etc., also can limit the depth of the search tree when using 
`-maxdepth` after the search start:
```
$ find /data2/junk -maxdepth 2 -type d -not -name "pipeline*"|head
/data2/junk
/data2/junk/.nextflow
/data2/junk/.nextflow/cache
/data2/junk/results
/data2/junk/results/Preprocessing
[...]
```
Have to remember, that `find` is not like `ls`, the order of file listing can look random, to make it alphabetical have 
to pipe it into `sort`. The `find` command is very powerful, there are dozen of other uses, a good starting point can 
be the [Bash Cookbook](https://www.oreilly.com/library/view/bash-cookbook-2nd/9781491975329/) that I gave it to 
somebody some time ago...

### Is it the same? [^](#Useful-bash-and-other-tricks)
Comparing files, especially small textfiles always was and is a routine UNIX task; de main tool used for this is `diff`, 
for example it can easily find differences (i.e. mistypings) in a small poem:
```
$ cat poem
The way a crow
Shook down on me
The dust of snow
From a hemlock tree
$ cat poem.new
The way a crow
Shook down on me
The dust of snow
From a hemlok tree
$ diff poem poem.new 
4c4
< From a hemlock tree
---
> From a hemlok tree
```
It can be difficult to find the missing *c* letter in the last line, but `diff` gives back the differing lines. 
Nevertheless, when the difference is huge, results from `diff` can be frightening. A similar tool, `sdiff` prints 
out both files, and puts a `|` sign to the line where they are differing (or a `<` or `>` if lines are missing from 
the first or from the second file respectively): 

```
$ sdiff poem poem.new 
The way a crow							The way a crow
Shook down on me						Shook down on me
The dust of snow						The dust of snow
From a hemlock tree				    |	From a hemlok tree
```  
Or, you can ask it to print out only the differing lines:

```
$ sdiff -s poem poem.new 
From a hemlock tree					|	From a hemlok tree
```
That is all nice and clean, but what to do with files that are >100G and were transferred via an unreliable network
connection? As an example, I have two files, exactly the same size:

```
 $ ls -l foo*
-rw-rw-r-- 1 szilva btb 1234321 Nov 19 19:19 foo1
-rw-rw-r-- 1 szilva btb 1234321 Nov 19 19:18 foo2
``` 
The `md5sum` utility will create an [MD5 hash](https://en.wikipedia.org/wiki/MD5) that can be used for checking data
integrity. This way one can compare large binary files: 
```
$ md5sum foo*
0635c824c0dce3aac4821be1bc81eae4  foo1
bebbb7f019e2060d37a577e6b1f41837  foo2
```
The funny sequence at the beginning is the MD5 hash, this should to be exactly the same for the two files, if they are
identical. A single byte difference will create a very different hash pattern. This hash is used by NGI when delivering 
data, the expected MD5 values are also delivered. For the 101 sample of the P13713 project the MD5 sums are:
```
szilva@munin /data0/P13713_fastq $ cat  P13713_101.md5
5578952f4f234377664be2da0fd9b8ff  P13713_101/02-FASTQ/190918_A00621_0127_BHMHCMDSXX/P13713_101_S29_L003_R1_001.fastq.gz
63fb6753320288f5c7f44bee67747bd9  P13713_101/02-FASTQ/190918_A00621_0127_BHMHCMDSXX/P13713_101_S29_L003_R2_001.fastq.gz
fc93efccabd89846d904bc2b7851ca87  P13713_101/02-FASTQ/191004_A00187_0202_BHNF7GDSXX/P13713_101_S54_L001_R2_001.fastq.gz
63a7943d871ab64174153b02ea069f49  P13713_101/02-FASTQ/191004_A00187_0202_BHNF7GDSXX/P13713_101_S54_L001_R1_001.fastq.gz
88e88d1e7af404b7eb5c85503849b144  P13713_101/02-FASTQ/190925_A00187_0199_AHN7J7DSXX/P13713_101_S1_L004_R1_001.fastq.gz
579a0bc0c93362eb6d6ff759004e6678  P13713_101/02-FASTQ/190925_A00187_0199_AHN7J7DSXX/P13713_101_S1_L004_R2_001.fastq.gz
```
If we calculate these on munin, the values are the same (well, for many files this should be done by `diff` and not 
by eye):
```
$ for f in `awk '{print $2}' P13713_101.md5`; do md5sum $f; done
5578952f4f234377664be2da0fd9b8ff  P13713_101/02-FASTQ/190918_A00621_0127_BHMHCMDSXX/P13713_101_S29_L003_R1_001.fastq.gz
63fb6753320288f5c7f44bee67747bd9  P13713_101/02-FASTQ/190918_A00621_0127_BHMHCMDSXX/P13713_101_S29_L003_R2_001.fastq.gz
fc93efccabd89846d904bc2b7851ca87  P13713_101/02-FASTQ/191004_A00187_0202_BHNF7GDSXX/P13713_101_S54_L001_R2_001.fastq.gz
63a7943d871ab64174153b02ea069f49  P13713_101/02-FASTQ/191004_A00187_0202_BHNF7GDSXX/P13713_101_S54_L001_R1_001.fastq.gz
88e88d1e7af404b7eb5c85503849b144  P13713_101/02-FASTQ/190925_A00187_0199_AHN7J7DSXX/P13713_101_S1_L004_R1_001.fastq.gz
579a0bc0c93362eb6d6ff759004e6678  P13713_101/02-FASTQ/190925_A00187_0199_AHN7J7DSXX/P13713_101_S1_L004_R2_001.fastq.gz

```

### All I want is fusion [^](#Useful-bash-and-other-tricks)
Consider you have an annotated VCF file that is usually pretty large with long lines. There are a handful possible 
fusions in the file, and you want to have a look only them, and want to filter out the other calls. You still have to 
retain the VCF header, so you have to have something that can do filter both. As many times `awk` can do it for you like:

```
awk '/^#/ || /gene_fusion/{print}' BigFile.vcf > fusions_only.vcf
```  
The pattern is between the `/  /` signs, `^` means "beginning of line", so, all VCF header lines that are starting with 
'#' will be printed out. Furthermore, if the line contains the "gene_fusion" string as annotation, that will be 
printed out also.  
 

### BAM subset [^](#Useful-bash-and-other-tricks)

It can be inconvenient to use all the huge BAM files all the time. To share data, or to have a look at the most 
important variants only it is advised to generate a subset. `samtools` can be used for this - if we have only a handful
of intervals, we can generate a subset from command-line. Have to have an indexed BAM:

```
$ samtools view -@32 -b -M -o withM.bam test.bam chr1:123456-234567 chr1:125000-456789
$ samtools view -@32 -b -o withoutM.bam test.bam chr1:123456-234567 chr1:125000-456789
$ ls -l 
-rw-rw-r-- 1 szilva btb 111238514776 Nov 17 18:23 test.bam
-rw-rw-r-- 1 szilva btb      9400672 Nov 17 18:19 test.bam.bai
-rw-rw-r-- 1 szilva btb     11048646 Nov 24 15:04 withM.bam
-rw-rw-r-- 1 szilva btb     15230493 Nov 24 15:05 withoutM.bam
```

Note the extra flags, and the intervals, that are overlapping. The `-@32 -b -o xxx.bam` stands for the number of
CPUs used, the output format (binary BAM), and output filename respectively. The intervals can be added in IGV format
as well (like `chr1:123.456-234.345` ), copied directly from IGV. The most important part is the `-M` flag, that 
makes sure we are using the multi-region iterator, that according to the manual "increases the speed, removes 
duplicates and outputs the reads as they are ordered in the file". As a result, if we are providing overlapping 
intervals, reads that are present in both interval will be written out only once. If the original BAM is sorted, the 
subset will be sorted as well. Hence the difference in the two BAM files above. The one without the `-M` flag 
(`withoutM.bam`) is bigger, because due to overlapping intervals many reads were written out twice.

That is all nice, but sometimes we want to have much more locations, i.e. all the important genes, or all the
prioritized variations. In that case the `-L` option helps, providing we have a BED file with all the locations. 
All the reads that are overlapping to the BED (even if with only a handful of bases) will be written out, and to be
safe always use the `-M` flag:

```
$ samtools view -@32 -M -b -o manysmall.bam -L intervals.bed test.bam
``` 

### How many of them? [^](#Useful-bash-and-other-tricks)

The basic command to count lines is `wc` (a.k.a. "word count")  that can do line counting  as well:
```
$ rpm -qa|wc -l
1830
```

The `rpm` utility is a package-manager for CentOS though better to use the more intelligent `yum` software. Still, this
short sample shows we have 1830 packages installed at the time of writing this paper on munin. We can also estimate
how many common variants were thrown out by VEP:

```
$ wc -l mutect2_P2233_101T_vs_P2233_120N.AF.snpEff.ann.vep.ann.vcf
111614 mutect2_P2233_101T_vs_P2233_120N.AF.snpEff.ann.vep.ann.vcf
$ wc -l mutect2_P2233_101T_vs_P2233_120N.AF.snpEff.ann.vcf 
113195 mutect2_P2233_101T_vs_P2233_120N.AF.snpEff.ann.vcf
``` 

Looks only 113195-111614=1581 lines are thrown out by VEP as common entries in this case. To count the occurrence of
a pattern the easiest is to use `grep -c` . We are using [find](#Where-is-my-file-) to scan through all the 
directories below, ignore `tmp` dirs, print out filenames and the number of lines that are matching to the
"gene_fusion" pattern.
  
```
$ for f in `find . -type f -name Manta*.somaticSV.vcf.snpEff.ann.vcf| grep -v tmp`; \
> do echo -n $f" " ; grep -c gene_fusion $f; done
./P2233_101T_P2233_120N/Annotation/SnpEff/Manta_P2233_101T_vs_P2233_120N.somaticSV.vcf.snpEff.ann.vcf 149
./P2233_102T_P2233_121N/Annotation/SnpEff/Manta_P2233_102T_vs_P2233_121N.somaticSV.vcf.snpEff.ann.vcf 9
./P2233_106T_P2233_125N/Annotation/SnpEff/Manta_P2233_106T_vs_P2233_125N.somaticSV.vcf.snpEff.ann.vcf 76
./P2233_103T_P2233_122N/Annotation/SnpEff/Manta_P2233_103T_vs_P2233_122N.somaticSV.vcf.snpEff.ann.vcf 15
./P2233_104T_P2233_123N/Annotation/SnpEff/Manta_P2233_104T_vs_P2233_123N.somaticSV.vcf.snpEff.ann.vcf 43
[...]
``` 

Indeed, `awk` and `perl` are much more powerful in pattern-search, but for most of the cases grep is just fine. We 
will see a shorter version of the loop going through the files using [xargs](#too-many-arguments-xargs-magic-) below. 

### run a script and log its output [^](#Useful-bash-and-other-tricks)

All commands in the shell have standard channels, most notably the standard output and the standard error (stdout
and stderr). We have experienced many times that some commands are writing the help text to the standard error, 
and we can not catch it with `less`. When we are trying to scroll in the GATK help page:

```
(gatk4) $ gatk Mutect2| less
~
~
```  

We are getting only tilde signs, instead of the help text. The solution is to 
[redirect](https://www.gnu.org/software/bash/manual/html_node/Redirections.html) the standard error to standard output,
like:

```
(gatk4) $ gatk Mutect2 2>&1| less
Using GATK jar /home/szilva/miniconda38/envs/gatk4/share/gatk4-4.1.9.0-0/gatk-package-4.1.9.0-local.jar
Running:
    java -Dsamjdk.use_async_io_read_samtools=false -Dsamjdk.use_async_io_write_samtools=true [...] 
USAGE: Mutect2 [arguments]
Call somatic SNVs and indels via local assembly of haplotypes
Version:4.1.9.0
Required Arguments:
--input,-I <GATKPath> ...  
```  

Therefore, if we have a script that runs for a long time, and want to log both its output and its errors, we can 
redirect these into a single file. The format is a bit weird though:

```
$ script_that_runs_for_ages.sh > output 2>&1
``` 

The drawback is that we can not see directly what is to be written into the file, I prefer to use a format like:
```
$ script_that_runs_for_ages.sh 2>&1 | tee whatever.`date +%Y-%m-%d-%Hh%Mm`.log
```
This will result in a logfile that contains the time of launching _and_ we can see on the terminal as well what is 
happening in the moment. The `tee` utility is to save stdout to a file and show it on the terminal at the same time. 

### What chromosomes are in this bloody large FASTA? [^](#Useful-bash-and-other-tricks)
Though grep would be the first choice to print out all the header lines from a FASTA file, awk has the advantage that
we can include fields (or columns) and in general easier to write short programs in awk. The short answer for the 
question above is like:  

```
$ awk '/>/{print $1}' Homo_sapiens_assembly38.fasta
>chr1
>chr2
>chr3
[...]
```

Furthermore, we can do other filtering with awk, i.e. generating a BED file from VCF (that we can feed into 
samtools to get a [subset of a large BAM](#bam-subset-)) with locations that are around the breakpoints of 
a reported fusions on chromosome 7:
```
$ awk '/gene_fusion/ && $1~/chr7/ {print $1"\t"$2-1000"\t"$2+1000}' Manta.somaticSV.vcf.snpEff.ann.vcf
chr7    6025350 6027350
chr7    6025643 6027643
chr7    27205805        27207805
[...]
chr7    55096739        55098739
chr7    100972593       100974593
chr7    119717942       119719942
```

the `/gene_fusion/ && $1~/chr7/` part means that those lines are matching where there is a "gene_fusion" mentioned 
and the first column (`$1`) also matches (`~`) the "chr7" string. The `{print $1"\t"$2-1000"\t"$2+1000}` part prints 
out the first column (the chromosome this case) and &#177;1000 bps coordinates from the breakpoint.  

### htop, stopping nextflow and all of my runaway stuff [^](#Useful-bash-and-other-tricks)

The `top` utility's younger brother is `htop`, that can show a better state about the server, i.e. printing out CPU
states, and is more convenient in general. It is important to have a look at its output even _after_ we have stopped
a nextflow process, since some software just simply do not want to die. It can happen that we realize we forget 
something from the command-line, want to add it, and restart the whole business, but looking at htop we can see that
the - supposedly - interrupted workflow steps are still running happily. So, we must kill them somehow, but first 
let's find them. The best to print them out with `ps` - it is a pretty complex command with many possible options. To 
print out all of our processes try:

```
$ ps xw -u szilva
   PID TTY      STAT   TIME COMMAND
   485 pts/10   Ss     0:06 /bin/bash
 22736 ?        Ss     0:32 SCREEN
 30486 ?        S      0:00 /bin/sh /home/szilva/.Xclients
 30592 ?        Ss     0:32 /usr/bin/dbus-daemon --fork --print-pid 5 --print-address 7 --session
 30641 ?        Ss     1:19 /usr/bin/ssh-agent /home/szilva/.Xclients
 30642 ?        Sl     0:57 mate-session
 30660 ?        Sl     0:00 /usr/libexec/at-spi-bus-launcher
 30665 ?        S      0:01 /usr/bin/dbus-daemon --config-file=...
 30741 ?        Sl   157:30 caja
 30742 ?        Sl     2:18 mate-screensaver
[...]
```
Immediately we can see there are zillion processes we are not aware running under our name. To filter out the rouge 
ones we have to have match a pattern, get the PID (process ID) and kill the process

```
ps xw -u szilva| awk '/name_I_do_not_like/{print $1}'|xargs kill
```  

There are (at least) two things in this line that are confusing: I did not write an exact process name, since one 
have to experiment him/herself which are the bad processes. The other part is the 
[xargs](#too-many-arguments-xargs-magic-) thingy that leads us to the next entry.   

### Too many arguments, xargs magic [^](#Useful-bash-and-other-tricks)

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore 
magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo 
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. 
Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.


### Get the filename only, or full path [^](#Useful-bash-and-other-tricks)

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore 
magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo 
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. 
Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.


### What env do I have? [^](#Useful-bash-and-other-tricks)

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore 
magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo 
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. 
Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

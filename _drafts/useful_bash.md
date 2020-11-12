---
title: 'Useful bash tricks '
date: 2020-11-08
permalink: /posts/2020/11/Useful_bash/
tags:
  - bash
  - linux
---

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
`-profile munin` directive. Besides this, there are quite a few configs available, including UPPMAX and AWS 
(https://github.com/nf-core/configs) . 

#### Sarek use cases:
There is already a collection of use cases in the documentation:  



### Reproduce things with conda
Many times you want to install some software, but you not want to bother the system manager. 
Or, your belowed software needs version 3.4.5 of some other software, but you have 

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





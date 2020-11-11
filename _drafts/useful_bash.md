---
title: 'Useful bash tricks '
date: 2020-11-08
permalink: /posts/2020/11/Useful_bash/
tags:
  - bash
  - linux
---
### Where is my file
You have 200T data around and you are remembering only vaguely the name of the file

### Is it the same
Compare files
- md5sum
- diff, sdiff

### How many of them
- wc -l
- grep -c

### reproduce things with singularity
singularity shell

### reproduce things with conda
conda bugger

### if you are still interested in modules ...
will write if have time

### yum is your friend

### run a script and log its output

./runBugger.sh 2>&1 | tee soffer.`date +%Y-%m-%d-%Hh%Mm`.log

### What chromosomes are in this bloody large FASTA?
awk '/>/{print}' /data1/references/igenomes/Homo_sapiens/GATK/GRCh38/Sequence/WholeGenomeFasta/Homo_sapiens_assembly38.fasta

### htop stopping nexftlow and all of my runaway stuff

### too many arguments, xargs magic

### basename readlink

### what env do I have?

### if not vim, use jed and Esc-f




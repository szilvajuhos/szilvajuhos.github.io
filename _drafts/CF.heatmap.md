---
title: 'Generating Somatic Mutations For NA12878'
date: 2020-06-08
permalink: /posts/2020/07/CNVheatmap/
tags:
  - sarek
  - control-freec
  - ascat
  - CNV
---



```
for b in data/P*freectobed.bed; do SAMPLE=`basename ${b%.hg38*}`; echo $SAMPLE;for f in `seq 1 22` X Y; do awk '/chr'$f'/{print}' ${b} > data/chr${f}/${SAMPLE}.chr${f}.bed; done; done

for f in `ls data/chr1/*bed| sort`;do SAMPLE=`basename ${f%.chr*}`; echo -n $SAMPLE"," && python it.py -b $f -l 248956422; done > chr1.csv
```

## CHR1
<br/><img src='/images/CFheatmap/chr1.png'>
## CHR2
<br/><img src='/images/CFheatmap/chr2.png'>
## CHR3
<br/><img src='/images/CFheatmap/chr3.png'>
<br/><img src='/images/CFheatmap/chr4.png'>
<br/><img src='/images/CFheatmap/chr5.png'>
<br/><img src='/images/CFheatmap/chr6.png'>
<br/><img src='/images/CFheatmap/chr7.png'>
<br/><img src='/images/CFheatmap/chr8.png'>
<br/><img src='/images/CFheatmap/chr9.png'>
<br/><img src='/images/CFheatmap/chr10.png'>
<br/><img src='/images/CFheatmap/chr11.png'>
<br/><img src='/images/CFheatmap/chr12.png'>
<br/><img src='/images/CFheatmap/chr13.png'>
<br/><img src='/images/CFheatmap/chr14.png'>
<br/><img src='/images/CFheatmap/chr15.png'>
<br/><img src='/images/CFheatmap/chr16.png'>
<br/><img src='/images/CFheatmap/chr17.png'>
<br/><img src='/images/CFheatmap/chr18.png'>
<br/><img src='/images/CFheatmap/chr19.png'>
<br/><img src='/images/CFheatmap/chr20.png'>
<br/><img src='/images/CFheatmap/chr21.png'>
<br/><img src='/images/CFheatmap/chr22.png'>
<br/><img src='/images/CFheatmap/chrX.png'>
<br/><img src='/images/CFheatmap/chrY.png'>

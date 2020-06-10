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

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



```for f in `cat current.samples`; do echo $f; for c in `seq 1 22` X Y;do CHROM="chr"${c};echo $CHROM; echo -n `basename -s .hg38.pileup.gz_ratio.freectobed.bed $f`",">>${CHROM}.csv; python it.py -b $f -c $CHROM -i ~/genome/Homo_sapiens_assembly38.fasta.fai >> ${CHROM}.csv;done; done
```

## CHR1:
<br/><img src='/images/CFheatmap/chr1.png'>

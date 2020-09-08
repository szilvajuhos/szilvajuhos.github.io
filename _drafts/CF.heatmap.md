---
title: 'Heatmap from Control-FREEC results'
date: 2020-06-08
permalink: /posts/2020/07/CNVheatmap/
tags:
  - sarek
  - control-freec
  - ascat
  - CNV
---



```
for f in `cat current.samples`; do echo $f; for c in `seq 1 22` X Y;do CHROM=chr${c}; echo -n $f",">>${CHROM}.csv; python it.py -b BedGraphs/${f}.hg38.pileup.gz_ratio.BedGraph -c $CHROM -i ~/genome/Homo_sapiens_assembly38.fasta.fai >> ${CHROM}.csv;done; done


```


Control-FREEC output:
 - \_CNVs: file with coordinates of predicted copy number alterations. 
 - \_ratio.txt: file with ratios and predicted copy number alterations for each window.
 - \_BAF.txt: file B-allele frequencies for each possibly heterozygous SNP position.
 - \_sample.cnp and \_control.cnp files: files with raw copy number profiles.
 - \_ratio.BedGraph  file with ratios in BedGraph format for visualization
 
 

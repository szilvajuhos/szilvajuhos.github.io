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

Control-FREEC output [as at their page](http://boevalab.inf.ethz.ch/FREEC/tutorial.html#OUTPUT):
 - \_CNVs: file with coordinates of predicted copy number alterations. 
 - \_ratio.txt: file with ratios and predicted copy number alterations for each window.
 - \_BAF.txt: file B-allele frequencies for each possibly heterozygous SNP position.
 - \_sample.cnp and \_control.cnp files: files with raw copy number profiles.
 - \_ratio.BedGraph  file with ratios in BedGraph format for visualization
 

### Scripts to generate different heatmaps are at [BTB-scripts repo](https://github.com/szilvajuhos/btb-scripts/blob/master/heatmap/) **

Finding files:
```
find /data1 /data0 -maxdepth 4 -type d| grep -f current.samples | grep Results_hg38| grep -v old
```
 
[makeBedGraphHeatmap.sh](https://github.com/szilvajuhos/btb-scripts/blob/master/heatmap/makeBedGraphHeatmap.sh)

Heatmap from BedGraph file (containing ratios) 
<br/><img src='/images/heatmap_BedGraph.png'>

Using ratios file and raw data the landscape is flat, because some ratio values are extreme:
<br/><img src='/images/heatmap_RatiosLinear.png'>

Scaling the same picture down to 1-3:
<br/><img src='/images/heatmap_RatiosLinear_1_3.png'>

Using log scale for ratios:
<br/><img src='/images/heatmap_RatiosLog.png'>

Copy Number column from the ratios file:
<br/><img src='/images/heatmap_ratioCN.png'>

Copy Numbers from the \_CNVs file:
<br/><img src='/images/heatmap_CNVs.png'>

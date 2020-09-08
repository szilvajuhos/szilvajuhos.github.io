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

Control-FREEC output:
 - \_CNVs: file with coordinates of predicted copy number alterations. 
 - \_ratio.txt: file with ratios and predicted copy number alterations for each window.
 - \_BAF.txt: file B-allele frequencies for each possibly heterozygous SNP position.
 - \_sample.cnp and \_control.cnp files: files with raw copy number profiles.
 - \_ratio.BedGraph  file with ratios in BedGraph format for visualization
 

** Scripts to generate different heatmaps are at [BTB-scripts repo](https://github.com/szilvajuhos/btb-scripts/blob/master/heatmap/) **

 
# Generating heatmap from BedGraph file (containing ratios) 
[makeBedGraphHeatmap.sh](https://github.com/szilvajuhos/btb-scripts/blob/master/heatmap/makeBedGraphHeatmap.sh)
<br/><img src='/images/heatmap_BedGraph.png'>


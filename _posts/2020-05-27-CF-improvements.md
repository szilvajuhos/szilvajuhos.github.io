---
title: 'Running Control-FREEC with Sarek'
date: 2020-05-27
permalink: /posts/2020/05/control-freec-sarek/
tags:
  - CNV
  - Control-FREEC
  - structural variants
  - sarek
---


Running from duplicate marked files (getting pileups first)
------
```
nextflow run MaxUlysse/nf-core\_sarek --input duplicateMarked.tsv -profile munin --step Recalibrate --skip\_qc bamQC
nextflow run MaxUlysse/nf-core\_sarek --input duplicateMarked.tsv -profile munin --step Control-FREEC --skip\_qc bamQC
```


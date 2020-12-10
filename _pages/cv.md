---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* M.S. research chemist, Roland Eotvos University, Budapest, 1990

Work experience
======
* 2016-
	* Since the beginning of 2016 I was mostly involved in the development of a cancer NGS
	workflow called Sarek https://nf-co.re/sarek, doing the same tasks at different positions.
	Besides software development, I was involved in testing and validation on real patient
	WGS data for hundreds of samples
	* Administering the local Dell EMC/Compellent Storage node of ~200T
	* Department of Oncology-Pathology, Karolinska Institutet, Stockholm (Barntumorbanken)
	* Uppsala University, Dept. Cell and Molecular Biology
	* SciLifeLab Genomics Application Group, Stockholm

*	2009-2015
	* Omixon Biocomputing, Budapest, Hungary, Senior Scientist
		* HLA typing and MHC-related NGS projects
		* Testing and developing code for next-generation sequencing data
		* Validating new algorithms and infrastructure for finding genetic variations

*	2007-2009
	* EMBL-EBI European Bioinformatics Institute, Cambridge, UK
		* Software engineer, maintaining and developing the database code/schema responsible for
		the daily updates of the EMBL Nucleotide Sequence Database (EMBL-Bank
		http://www.ebi.ac.uk/ena/about/about)
 
*	2003-2007
	* Chemaxon Ltd, Budapest, Hungary
		* Software engineer, core developer for the flagship Marvin software
		https://chemaxon.com/products/marvin2001-2003

* 2001-2003
	* Ribotargets Ltd, Cambridge, UK (now: Vernalis plc)
		* Computational chemist
		* Structure based drug design, C++ code for docking
		* [rdock](https://github.com/rxdock/rxdock) is open-source now

* 2000-2001
	* Wolfson Brain Imaging Centre (University of Cambridge, UK)
		* UNIX system administration for the MRI and PET centre

* 1998-2000
	* University of Durham, Dept. Physics, Astronomy Instrumentation Group
	* Software engineer, astronomical telescope control
	* Managing local web servers and UNIX workstations

* 1990-1998
	* Balaton Limnological Research Institute of Hungarian Acad. Sci.:
		* Research Assistant,Modeling ecological/chemical processes related to alga blooms
		* Wet lab experience (HPLC, monoamine ligand-binding assays), C/C++, models for neural activity, 
		on-line data acquisition, statistical modeling of data, time series prediction, artificial 
		neural networks
		* Supervising the local computer network

* 1985-90
	* M.Sc. student at the Roland Eotvos University
		* Thesis: Computerized Monte-Carlo Simulation for Gelation of Microphases, Written in 
		C on a VAX/VMS system
 
Skills
======

* Bioinformatics (NGS alignment tools, cancer WGS pipeline)
* HLA typing (the bioinformatics part)
* Object-oriented programming (C/C++ , Java, Python)
* Version control (git, svn, csv, perforce)
* Scripting ( Bash, Perl )
* Molecular modelling ( MOE /SVL, Chemaxon tools)
* DB administration/development (mainly Oracle, SQL)
* UNIX system management in heterogeneous environment (storage/HPC/RAID/cold backup)
* Data mining, pattern recognition


Publications
======
  <ul>{% for post in site.publications %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
  

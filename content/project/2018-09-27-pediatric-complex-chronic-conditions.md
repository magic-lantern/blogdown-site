---
title: R Package for Pediatric Complex Chronic Condition Classification
author: Seth Russell
date: '2018-09-27'
categories:
  - R
tags:
  - R
  - project
slug: pediatric-complex-chronic-conditions
---

## Motivation
Version 2 of the Pediatric Complex Chronic Conditions (CCC) system was published 
["Pediatric complex chronic conditions classification system version
2: updated for ICD-10 and complex medical technology dependence and
transplantation" by Chris Feudtner, James A Feinstein, Wenjun Zhong, Matt Hall
and Dingwei Dai](http://bmcpediatr.biomedcentral.com/articles/10.1186/1471-2431-14-199).

SAS and STATA scripts to generate CCC categories from ICD codes were provided by Feudtner et al. 
as an appendix to the above manuscript. However, those scripts can take many hours to run
on large datasets. 

This package provides R functions to generate the CCC categories. Because the R functions
are built with a C++ back-end, they are very computationally efficient.

## Installation

### From CRAN
Version 1.0.2 is available on The Comprehensive R Archive Network at https://CRAN.R-project.org/package=pccc.


### Developmental version

You can install the
developmental version of `pccc` directly from github using the 
[`devtools`](https://github.com/hadley/devtools/) package:

    if (!("devtools" %in% rownames(installed.packages()))) { 
      warning("installing devtools from https://cran.rstudio.com")
      install.packages("devtools", repo = "https://cran.rstudio.com")
    }

    devtools::install_github("CUD2V/pccc", build_vignettes = TRUE)

*NOTE:* If you are working on a Windows machine you will need to download and
install [`Rtools`](https://cran.r-project.org/bin/windows/Rtools/) before
`devtools` will work for you.

If you are on a Linux machine or have GNU `make` configured you should be able
to build and install this package by cloning the repository and running

    make install

## More project information

We've published a research letter in JAMA Pediatrics on this R package:

https://jamanetwork.com/journals/jamapediatrics/fullarticle/2677901

For source code see:

https://github.com/CUD2V/pccc

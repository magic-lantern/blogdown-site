---
title: Kung Faux Pandas
author: Seth Russell
date: '2018-10-04'
categories:
  - Python
tags:
  - phi
  - python
  - synthetic data
  - project
slug: kung-faux-pandas
description: A data synthesis tool to simplify privacy preservation
---

There are many barriers to data access and data sharing, especially in the domain of machine learning on health care data. Legal constraints such as HIPAA protect patient privacy but slow access to data and limit reproducibility. Kung Faux Pandas is an end-to-end system called Kung Faux Pandas for easily generating de-identified or synthetic data which is statistically similar to given real data but lacks sensitive information. This system focuses on data synthesis and de-identification narrowed to a specific research question to allow for self-service data access without the complexities required to generate an entire population of data that is not needed for a given research project. Kung Faux Pandas is an open source publicly available system that lowers barriers to HIPAA- and GDPR-compliant data sharing for enabling reproducibility and other purposes.

This data synthesis process was created using Python. In order to make it easier for others to replicate the environment, a range of environments are supported: Conda environment export, virtualenv requirements file, and a docker image with the latest code.

For source code see:

https://github.com/CUD2V/kungfauxpandas


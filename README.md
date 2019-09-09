# Apache Cassandra on Azure VMs Performance Experiments
September 2019

## Overview
Many customers run Apache Cassandra in Azure and are looking for experiment-based guidance for tuning Cassandra on Azure VMs. This repo summarizes learnings from performing various relative performance tests of Apache Cassandra on different Azure VM configurations to answer a few common questions such as: 
* What stripe/chunk size should I use for Cassandra data disks?
* What is the impact of data disk caching? 
* Will my cluster performance scale linearly if I add another Cassandra data center to my ring in another Azure region?
* Is there a performance difference between `ext4` and `xfs` filesystems?

The goal of this repo is to share interesting learnings to help increase knowledge around how Apache Cassandra will behave and perform on various Azure VM configurations. It is not meant to be used as a benchmark of Cassandra on Azure, but rather as a summary of observations and conclusions from micro-scale tests comparing relative performance observed when tuning Azure VM and Cassandra configurations.


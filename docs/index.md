---
title: "Developing WDL Workflows"
date: "June, 2024"
site: bookdown::bookdown_site
documentclass: book
bibliography: [book.bib]
biblio-style: apalike
link-citations: yes
description: "Description about Course/Book."
favicon: assets/dasl_favicon.ico
output:
    bookdown::word_document2:
      toc: true
---

# About this Course {-}

"Developing WDL Workflows" shows a bioinformatics workflow developer how to strategically develop and scale up a WDL workflow that is iterative, reproducible, and efficient in terms of time and resource used. This guide is flexible regardless of where the data is, what computing resources are being used, and what software is being used. 

## Target Audience

The course is intended for first time developers of the [WDL workflow language](https://github.com/openwdl/wdl), who wants to iteratively develop a WDL bioinformatics workflow. In order to use this guide the audience should be able to comprehend introductory WDL syntax, and should be able to run a WDL workflow on a computing engine of their choice, such as Cromwell, miniwdl, or a cloud computing environment such as Terra, AnVIL, or Dockstore. 

### Relevant Resources

If you are new to WDL, [OpenWDL Docs](https://docs.openwdl.org/en/stable/) is an excellent WDL companion resource to help you get started and is platform agnostic. OpenWDL Docs focuses on the basic grammar of WDL as well as providing excellent cookbook recipes of common WDL workflow structures. In this guide we will reference these basic grammar structures and common workflow cookbook recipes.

## Why WDL?

You may have encountered other workflow tools, such as Snakemake or Nextflow, and those are highly capable. Why learn a brand new workflow language? Let's review some WDL Pros and Cons:

### WDL Pros

WDL has some really helpful advantages compared to other frameworks:

- **Portability**. WDL can run on nearly any system, whether it be your local computer, or on an HPC cluster, or on the Cloud, with platforms such as DNAnexus. In fact, a lot of developers will prototype a WDL workflow on their own local computer before moving it to the cloud.

- **Reproducibility**. Ever have the headache of having to reproduce the exact package versions to get your workflow to work again? If you use Docker containers to specify your software environment, you do not have to worry about this headache. A workflow will run identically locally, on HPC, or the cloud.

- **Sharing**. A WDL workflow is much easier to share with colleagues and is a good way to get credit for work you do everyday. If you spent time building it, why not share it? WDL is also an open standard and supported by a number of software tools.

- **Running and Making WDL workflows is a transferable skill.** Genomics and Pharma companies rely on WDL workflows to process thousands of FASTA/VCF files for their studies. They need more experts. It makes you more hireable within both Academia and Industry. 

### WDL Cons

Of course, nothing is free. WDL does require you to understand the basic concepts and terminologies including:

 - Basics of Docker

 - Understanding the WDL framework

 - Converting your bash scripts into WDL tasks and workflows


## Curriculum


The course covers the following: 

- How to write an effective WDL task

- Link multiple WDL tasks together in a workflow 

- Organize variables via structs, scale multiple samples via Arrays

- Reuse repeated tasks via task aliasing

- Configure settings for the execution engine



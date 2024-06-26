

# Connecting multiple tasks together in a linear chain

Now that you have a first task in a workflow up and running, the next step is to continue building out the workflow as described in our Workflow Plan:

1.  `BwaMem` aligns the samples to the reference genome.
2.  `MarkDuplicates` marks PCR duplicates.
3.  `ApplyBaseRecalibrator` applies base recalibration.
4.  `Mutect2` performs somatic mutation calling. For this current iteration, we start with `Mutect2TumorOnly` which only uses the tumor sample. Later on, we will switch to a version that will allow for comparing tumor samples against a non-tumor normal sample.
5.  `annovar` annotates the called somatic mutations.

We do this via a **linear chain**, in which we feed the output of one task to the input of the next task. Let's see how to build a linear chain in a workflow.

## How to connect tasks together in a workflow

We can easily connect tasks together because of the following: *The output variables of a task can be accessed at the workflow level as inputs for the subsequent task.*

For instance, let's see the output of our `BwaMem` task.

```         
output {
    File analysisReadyBam = "~{base_file_name}.aligned.bam"
    File analysisReadySorted = "~{base_file_name}.sorted_query_aligned.bam"
  }
```

The File variables `analysisReadyBam` and `analysisReadySorted` can now be accessed anywhere in the workflow block after the `BwaMem` task as `BwaMem.analysisReadyBam` and `BwaMem.analysisReadySorted`, respectively. Therefore, when we call the `MarkDuplicates` task, we can pass it the input `BwaMem.analysisReadySorted` from the `BwaMem` task:


```         
workflow mutation_calling {
  input {
    File sampleFastq

    # Reference genome
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
  }
    

  #  Map reads to reference
  call BwaMem {
    input:
      input_fastq = sampleFastq,
      ref_fasta = ref_fasta,
      ref_fasta_index = ref_fasta_index,
      ref_dict = ref_dict,
      ref_amb = ref_amb,
      ref_ann = ref_ann,
      ref_bwt = ref_bwt,
      ref_pac = ref_pac,
      ref_sa = ref_sa
  }
   
  call MarkDuplicates {
    input:
      input_bam = BwaMem.analysisReadySorted
  }
}
```

::: {.notice data-latex="notice"}
Resources:

<ul>

<li>For a basic introduction to linear chain, see [OpenWDL Docs](https://docs.openwdl.org/en/latest/WDL/Linear_chaining/)' introduction. </li>

<li>To see other examples of linear chain and variations, see [OpenWDL Docs](https://docs.openwdl.org/en/latest/WDL/add_plumbing/)'s section on workflow plumbing. </li>

</ul>
:::




## Writing `MarkDuplicates` task

Of course, the task `MarkDuplicates` hasn't been written yet! Let's go through it together:

### Input

The task takes an input bam file that has been aligned to the reference genome. It needs to be a File `input_bam` based on how we introduced it in the workflow above. That is easy to write up:

```         
task MarkDuplicates {
  input {
    File input_bam
  }
}
```

### Private variables in the task

Similar to the `BwaMem` task, we will name our output files based on the base name of the original input file in the workflow. Therefore, it makes sense to create a private String variable `base_file_name` that contains this base name of `input_bam`. We will use `base_file_name` in the Command section to specify the output bam and metrics files.

```         
task MarkDuplicates {
  input {
    File input_bam
  }
  String base_file_name = basename(input_bam, ".sorted_query_aligned.bam")
}
```

### Command

As we have been doing all along, we refer to any defined variables from the input or private variables using \~{this} syntax.

```         
task MarkDuplicates {
  input {
    File input_bam
  }

  String base_file_name = basename(input_bam, ".sorted_query_aligned.bam")

  command <<<
    gatk MarkDuplicates \
      --INPUT "~{input_bam}" \
      --OUTPUT "~{base_file_name}.duplicates_marked.bam" \
      --METRICS_FILE "~{base_file_name}.duplicate_metrics" \
      --CREATE_INDEX true \
      --OPTICAL_DUPLICATE_PIXEL_DISTANCE 100 \
      --VALIDATION_STRINGENCY SILENT
  >>>
}
```

### Runtime and Output

We specify a different Docker image that contains the GATK software, and the relevant computing needs. We also specify three different output files, two of which are specified in the command section, and the third is a bam index file that is automatically created by the command section.

Below is the task all together. It has a form very similar to our first task `BwaMem`.

```         
 task MarkDuplicates {
  input {
    File input_bam
  }

  String base_file_name = basename(input_bam, ".sorted_query_aligned.bam")

  command <<<
    gatk MarkDuplicates \
      --INPUT "~{input_bam}" \
      --OUTPUT "~{base_file_name}.duplicates_marked.bam" \
      --METRICS_FILE "~{base_file_name}.duplicate_metrics" \
      --CREATE_INDEX true \
      --OPTICAL_DUPLICATE_PIXEL_DISTANCE 100 \
      --VALIDATION_STRINGENCY SILENT
  >>>
  
  runtime {
    docker: "broadinstitute/gatk:4.1.4.0"
    memory: "48 GB"
    cpu: 4
  }
  
  output {
    File markDuplicates_bam = "~{base_file_name}.duplicates_marked.bam"
    File markDuplicates_bai = "~{base_file_name}.duplicates_marked.bai"
    File duplicate_metrics = "~{base_file_name}.duplicate_metrics"
  }
}
```

### Testing the workflow

As before, when you add a new task to the workflow, you should always test that it works on your test sample. To check that `MarkDuplicates` is indeed marking PCR duplicates, you could check for the presence of the [PCR duplicate flag in reads](https://gatk.broadinstitute.org/hc/en-us/articles/360051306171-MarkDuplicates-Picard), which has a decimal value of 1024 in the SAM Flags Field.

## The rest of the linear chain workflow

We build out the rest of the tasks in a very similar fashion. Tasks `ApplyBaseRecalibrator` and `Mutect2TumorOnly` both have files that need to be localized, but otherwise all the tasks have a very similar form as `BwaMem` and `MarkDuplicates`. For this current iteration, we use only the tumor sample for mutation calling. In the following chapters, we will use additional WDL features to make use of tumor and normal samples for mutation calling. 

<script src="https://gist.github.com/fhdsl-robot/2773cdaca9c042d3455e3cab71d4b4ba.js"></script>

We also expand our input JSON metadata to have files needed for each task:

```
{
  "mutation_calling.sampleFastq": "/path/to/Tumor_2_EGFR_HCC4006_combined.fastq",
  "mutation_calling.ref_fasta": "/path/to/Homo_sapiens_assembly19.fasta",
  "mutation_calling.ref_fasta_index": "/path/to/Homo_sapiens_assembly19.fasta.fai",
  "mutation_calling.ref_dict": "/path/to/Homo_sapiens_assembly19.dict",
  "mutation_calling.ref_pac": "/path/to/Homo_sapiens_assembly19.fasta.pac",
  "mutation_calling.ref_sa": "/path/to/Homo_sapiens_assembly19.fasta.sa",
  "mutation_calling.ref_amb": "/path/to/Homo_sapiens_assembly19.fasta.amb",
  "mutation_calling.ref_ann": "/path/to/Homo_sapiens_assembly19.fasta.ann",
  "mutation_calling.ref_bwt": "/path/to/Homo_sapiens_assembly19.fasta.bwt",
  "mutation_calling.ref_name": "hg19",
  "mutation_calling.dbSNP_vcf_index": "/path/to/dbsnp_138.b37.vcf.gz.tbi",
  "mutation_calling.dbSNP_vcf": "/path/to/dbsnp_138.b37.vcf.gz",
  "mutation_calling.known_indels_sites_indices": "/path/to/Mills_and_1000G_gold_standard.indels.b37.sites.vcf.idx",
  "mutation_calling.known_indels_sites_VCFs": "/path/to/Mills_and_1000G_gold_standard.indels.b37.sites.vcf",
  "mutation_calling.af_only_gnomad": "/path/to/af-only-gnomad.raw.sites.b37.vcf.gz",
  "mutation_calling.af_only_gnomad_index": "/path/to/af-only-gnomad.raw.sites.b37.vcf.gz.tbi",
  "mutation_calling.annovar_protocols": "refGene,knownGene,cosmic70,esp6500siv2_all,clinvar_20180603,gnomad211_exome",
  "mutation_calling.annovar_operation": "g,f,f,f,f,f"
}
```


<details>
<summary><b>The JSON using the Fred Hutch HPC</b></summary>

```
{
  "mutation_calling.sampleFastq": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/workflow_testing_data/WDL/wdl_101/HCC4006_final.fastq",
  "mutation_calling.ref_fasta": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta",
  "mutation_calling.ref_fasta_index": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.fai",
  "mutation_calling.ref_dict": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.dict",
  "mutation_calling.ref_pac": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.pac",
  "mutation_calling.ref_sa": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.sa",
  "mutation_calling.ref_amb": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.amb",
  "mutation_calling.ref_ann": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.ann",
  "mutation_calling.ref_bwt": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Homo_sapiens_assembly19.fasta.bwt",
  "mutation_calling.ref_name": "hg19",
  "mutation_calling.dbSNP_vcf_index": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/dbsnp_138.b37.vcf.gz.tbi",
  "mutation_calling.dbSNP_vcf": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/dbsnp_138.b37.vcf.gz",
  "mutation_calling.known_indels_sites_indices": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Mills_and_1000G_gold_standard.indels.b37.sites.vcf.idx",
  "mutation_calling.known_indels_sites_VCFs": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/Mills_and_1000G_gold_standard.indels.b37.sites.vcf",
  "mutation_calling.af_only_gnomad": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/af-only-gnomad.raw.sites.b37.vcf.gz",
  "mutation_calling.af_only_gnomad_index": "/fh/fast/paguirigan_a/pub/ReferenceDataSets/genome_data/human/hg19/af-only-gnomad.raw.sites.b37.vcf.gz.tbi",
  "mutation_calling.annovar_protocols": "refGene,knownGene,cosmic70,esp6500siv2_all,clinvar_20180603,gnomad211_exome",
  "mutation_calling.annovar_operation": "g,f,f,f,f,f"
}
```

</details>


<iframe src="https://docs.google.com/forms/d/e/1FAIpQLSeEKGWTJOowBhFlWftPUjFU8Rfj-d9iXIHENyd8_HGS8PM7kw/viewform?embedded=true" width="640" height="886" frameborder="0" marginheight="0" marginwidth="0">Loading…</iframe>

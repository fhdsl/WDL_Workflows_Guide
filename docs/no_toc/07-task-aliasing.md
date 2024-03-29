

# Task Aliasing

We've already gone over running a task multiple times in the context of scattered tasks. However, you may also want a task to run more than one time if that task is to run on multiple sets of inputs. In our case, we want to run a similar analysis on tumor samples and samples taken from normal tissue.

WDL has a sophisticated feature that allows one to reuse the same task repeatedly through your workflow: **task aliasing**
Simply put **task aliasing**  allows for the re-use of task definitions within the same workflow under different names, or "aliases". 

## Advantages of aliasing

The major advantages of using task aliasing are:

**1. Reduces Redundancy**: You don't need to copy and paste the same task definition multiple times and your workflows will be more concise and organized. 

**2. Simplifies Maintenance** : If you decide to change/update/fix a task, using task aliasing will make life easy as you need to update only once in your workflow. 

**3. Enhances Readability and Clarity**: A shorter workflow is easier to read but task aliasing also helps to contextualize the workflow ( for example are you doing this task for Sample set A or Sample set B)

**4. Facilitates Modular Workflow Design**: Task aliasing help to make your workflow modular. This is easier to adopt by 

**5. Improves Workflow Scalability**: Using task aliasing it is much easier to scale the workflow across different inputs. For example you want to run a task on different sample groups (Sample set A and B) will allow the same task to be run parallely and if you choose with different modifications. 

**6. Ensures Consistency**: Task aliasing assures that there is consistency in replicated tasks and helps the reader easily identify where changes are expected in a task. 

## Start to add a task alias

You can only alias a task that is already defined, so we will start with the BwaMem task rather than writing a new one.

> Note: In the real world, typically two samples would be processed from a patient: One tumor and one normal. However, we are writing a workflow that only takes in one normal sample and multiple tumor samples. This implies that we have taken multiple tumor samples from the same patient, and we're comparing all of them against a single normal sample.

Here we are creating an alias for the task BwaMem. We want to do this so it can run this task on the "normal" samples and store them seperately. 

First, make sure that in your workflow input, you reference to the normal samples as input.

```
workflow mutation_calling {
  input {
    ...
    File normalSamples
...
  }
```

Next all that you need to do is `call` the `task` you want to alias and use `as` to the `alias_of_your_choice`. 

But don't forget to make sure that all the inputs reflect actually different things we want to run this task on.

In this case we will be using a different sample and therefore the input_fastq is directed to the appropriate file source. 

## Add the task alias

```
  call BwaMem as normalBwaMem {
    input:
      input_fastq = normalSamples,
      refGenome = refGenome
  }
```

## Modify your output

 And finally you will also want to make sure that in your outputs section you are saving the appropriate outputs to reflect the task alias. 

```
output {
File normalalignedBamSorted = normalBwaMem.analysisReadySorted
}
```

## Alias for other tasks

We can do this for the other two tasks in our workflow as well

```
call MarkDuplicates as normalMarkDuplicates {
    input:
      input_bam = normalBwaMem.analysisReadySorted
  }

  call ApplyBaseRecalibrator as normalApplyBaseRecalibrator {
    input:
      input_bam = normalMarkDuplicates.markDuplicates_bam,
      input_bam_index = normalMarkDuplicates.markDuplicates_bai,
      dbSNP_vcf = dbSNP_vcf,
      dbSNP_vcf_index = dbSNP_vcf_index,
      known_indels_sites_VCFs = known_indels_sites_VCFs,
      known_indels_sites_indices = known_indels_sites_indices,
      refGenome = refGenome
  }
```

Now adding these steps to the workflow we will have our tumor and normal sample aligned and recalibrated and suitable for ingestion into the mutation calling step for a paired mutation calling using MuTect2. 

```

# The full workflow with task alias

version 1.0
## WDL 101 example workflow
## 
## This WDL workflow is intended to be used along with the WDL 101 docs. 
## This workflow should be used for inspiration purposes only. 
##
## We use three samples 
## Samples:
## MOLM13: Normal sample
## CALU1: KRAS G12C mutant
## HCC4006: EGFR Ex19 deletion mutant 
##
## Input requirements:
## - combined fastq files for chromosome 12 and 7 +/- 200bp around the sites of mutation only
##
## Output Files:
## - An aligned bam for all 3 samples (with duplicates marked and base quality recalibrated)
## 
## Workflow developed by Sitapriya Moorthi, Chris Lo and Taylor Firman @ Fred Hutch and Ash (Aisling) O'Farrell @ UCSC LMD: 02/28/24 for use @ Fred Hutch.

struct referenceGenome {
    File ref_fasta
    File ref_fasta_index
    File ref_dict
    File ref_amb
    File ref_ann
    File ref_bwt
    File ref_pac
    File ref_sa
    String ref_name
}


workflow mutation_calling {
  input {
    Array[File] tumorSamples
    File normalSamples

    referenceGenome refGenome
    
    File dbSNP_vcf
    File dbSNP_vcf_index
    File known_indels_sites_VCFs
    File known_indels_sites_indices
    
    File af_only_gnomad
    File af_only_gnomad_index
    
    String annovar_protocols
    String annovar_operation
  }
 
  # Scatter for "tumor" samples   
  scatter (tumorFastq in tumorSamples) {
    call BwaMem as tumorBwaMem {
      input:
        input_fastq = tumorFastq,
        refGenome = refGenome             ##pass in our struct
    }
    
    call MarkDuplicates as tumorMarkDuplicates {
      input:
        input_bam = tumorBwaMem.analysisReadySorted
    }

    call ApplyBaseRecalibrator as tumorApplyBaseRecalibrator{
      input:
        input_bam = tumorMarkDuplicates.markDuplicates_bam,
        input_bam_index = tumorMarkDuplicates.markDuplicates_bai,
        dbSNP_vcf = dbSNP_vcf,
        dbSNP_vcf_index = dbSNP_vcf_index,
        known_indels_sites_VCFs = known_indels_sites_VCFs,
        known_indels_sites_indices = known_indels_sites_indices,
        refGenome = refGenome
      }

    call Mutect2Paired {
    input:
      tumor_bam = tumorApplyBaseRecalibrator.recalibrated_bam,
      tumor_bam_index = tumorApplyBaseRecalibrator.recalibrated_bai,
      normal_bam = normalApplyBaseRecalibrator.recalibrated_bam,
      normal_bam_index = normalApplyBaseRecalibrator.recalibrated_bai,
      refGenome = refGenome,
      genomeReference = af_only_gnomad,
      genomeReferenceIndex = af_only_gnomad_index
  }

  call annovar {
    input:
      input_vcf = Mutect2Paired.output_vcf,
      ref_name = refGenome.ref_name,
      annovar_operation = annovar_operation,
      annovar_protocols = annovar_protocols
  }
}
  
  # Do for normal sample
  call BwaMem as normalBwaMem {
    input:
      input_fastq = normalSamples,
      refGenome = refGenome
  }
  
  call MarkDuplicates as normalMarkDuplicates {
    input:
      input_bam = normalBwaMem.analysisReadySorted
  }

  call ApplyBaseRecalibrator as normalApplyBaseRecalibrator {
    input:
      input_bam = normalMarkDuplicates.markDuplicates_bam,
      input_bam_index = normalMarkDuplicates.markDuplicates_bai,
      dbSNP_vcf = dbSNP_vcf,
      dbSNP_vcf_index = dbSNP_vcf_index,
      known_indels_sites_VCFs = known_indels_sites_VCFs,
      known_indels_sites_indices = known_indels_sites_indices,
      refGenome = refGenome
  }


  output {
    Array[File] tumoralignedBamSorted = tumorBwaMem.analysisReadySorted
    Array[File] tumorMarkDuplicates_bam = tumorMarkDuplicates.markDuplicates_bam
    Array[File] tumorMarkDuplicates_bai = tumorMarkDuplicates.markDuplicates_bai
    Array[File] tumoranalysisReadyBam = tumorApplyBaseRecalibrator.recalibrated_bam 
    Array[File] tumoranalysisReadyIndex = tumorApplyBaseRecalibrator.recalibrated_bai
    File normalalignedBamSorted = normalBwaMem.analysisReadySorted
    File normalmarkDuplicates_bam = normalMarkDuplicates.markDuplicates_bam
    File normalmarkDuplicates_bai = normalMarkDuplicates.markDuplicates_bai
    File normalanalysisReadyBam = normalApplyBaseRecalibrator.recalibrated_bam 
    File normalanalysisReadyIndex = normalApplyBaseRecalibrator.recalibrated_bai
    Array[File] Mutect2Paired_Vcf = Mutect2Paired.output_vcf
    Array[File] Mutect2Paired_VcfIndex = Mutect2Paired.output_vcf_index
    Array[File] Mutect2Paired_AnnotatedVcf = annovar.output_annotated_vcf
    Array[File] Mutect2Paired_AnnotatedTable = annovar.output_annotated_table
  }
}
# TASK DEFINITIONS

# Align fastq file to the reference genome
task BwaMem {
  input {
    File input_fastq
    referenceGenome refGenome         ## Our struct as input
    Int threads = 16
  }
  
  String base_file_name = basename(input_fastq, ".fastq")
  String ref_fasta_local = basename(refGenome.ref_fasta)  ##refer to ref_fasta here in struct

  String read_group_id = "ID:" + base_file_name
  String sample_name = "SM:" + base_file_name
  String platform = "illumina"
  String platform_info = "PL:" + platform   # Create the platform information


  command <<<
    set -eo pipefail

    #can we iterate through a struct??
    mv ~{refGenome.ref_fasta} .
    mv ~{refGenome.ref_fasta_index} .
    mv ~{refGenome.ref_dict} .
    mv ~{refGenome.ref_amb} .
    mv ~{refGenome.ref_ann} .
    mv ~{refGenome.ref_bwt} .
    mv ~{refGenome.ref_pac} .
    mv ~{refGenome.ref_sa} .

    bwa mem \
      -p -v 3 -t ~{threads} -M -R '@RG\t~{read_group_id}\t~{sample_name}\t~{platform_info}' \
      ~{ref_fasta_local} ~{input_fastq} > ~{base_file_name}.sam 
    samtools view -1bS -@ 15 -o ~{base_file_name}.aligned.bam ~{base_file_name}.sam
    samtools sort -@ 15 -o ~{base_file_name}.sorted_query_aligned.bam ~{base_file_name}.aligned.bam
  >>>

  output {
    File analysisReadySorted = "~{base_file_name}.sorted_query_aligned.bam"
  }
  
  runtime {
    memory: "48 GB"
    cpu: 16
    docker: "ghcr.io/getwilds/bwa:0.7.17"
  }
}

# Mark duplicates (not SPARK, for some reason that does something weird)
task MarkDuplicates {
  input {
    File input_bam
  }

  String base_file_name = basename(input_bam, ".sorted_query_aligned.bam")
  String output_bam = "~{base_file_name}.duplicates_marked.bam"
  String output_bai = "~{base_file_name}.duplicates_marked.bai"
  String metrics_file = "~{base_file_name}.duplicate_metrics"

  command <<<
    gatk MarkDuplicates \
      --INPUT ~{input_bam} \
      --OUTPUT ~{output_bam} \
      --METRICS_FILE ~{metrics_file} \
      --CREATE_INDEX true \
      --OPTICAL_DUPLICATE_PIXEL_DISTANCE 100 \
      --VALIDATION_STRINGENCY SILENT
  >>>

  runtime {
    docker: "ghcr.io/getwilds/gatk:4.3.0.0"
    memory: "48 GB"
    cpu: 4
  }

  output {
    File markDuplicates_bam = "~{output_bam}"
    File markDuplicates_bai = "~{output_bai}"
    File duplicate_metrics = "~{metrics_file}"
  }
}

# Base quality recalibration
task ApplyBaseRecalibrator {
  input {
    File input_bam
    File input_bam_index
    File dbSNP_vcf
    File dbSNP_vcf_index
    File known_indels_sites_VCFs
    File known_indels_sites_indices
    referenceGenome refGenome         ## Use struct as input for task
  }
  
  String base_file_name = basename(input_bam, ".duplicates_marked.bam")
  
  String ref_fasta_local = basename(refGenome.ref_fasta)
  String dbSNP_vcf_local = basename(dbSNP_vcf)
  String known_indels_sites_VCFs_local = basename(known_indels_sites_VCFs)


  command <<<
  set -eo pipefail

  mv ~{refGenome.ref_fasta} .
  mv ~{refGenome.ref_fasta_index} .
  mv ~{refGenome.ref_dict} .

  mv ~{dbSNP_vcf} .
  mv ~{dbSNP_vcf_index} .

  mv ~{known_indels_sites_VCFs} .
  mv ~{known_indels_sites_indices} .

  samtools index ~{input_bam} #redundant? markduplicates already does this?

  gatk --java-options "-Xms8g" \
      BaseRecalibrator \
      -R ~{ref_fasta_local} \
      -I ~{input_bam} \
      -O ~{base_file_name}.recal_data.csv \
      --known-sites ~{dbSNP_vcf_local} \
      --known-sites ~{known_indels_sites_VCFs_local} \
      

  gatk --java-options "-Xms8g" \
      ApplyBQSR \
      -bqsr ~{base_file_name}.recal_data.csv \
      -I ~{input_bam} \
      -O ~{base_file_name}.recal.bam \
      -R ~{ref_fasta_local} \
      

  #finds the current sort order of this bam file
  samtools view -H ~{base_file_name}.recal.bam | grep @SQ | sed 's/@SQ\tSN:\|LN://g' > ~{base_file_name}.sortOrder.txt
>>>

  output {
    File recalibrated_bam = "~{base_file_name}.recal.bam"
    File recalibrated_bai = "~{base_file_name}.recal.bai"
    File sortOrder = "~{base_file_name}.sortOrder.txt"
  }
  runtime {
    memory: "36 GB"
    cpu: 2
    docker: "ghcr.io/getwilds/gatk:4.3.0.0"
  }
}

# Mutect 2 calling tumor-normal

task Mutect2Paired {
  input {
    File tumor_bam
    File tumor_bam_index
    File normal_bam
    File normal_bam_index
    referenceGenome refGenome           # our struct as input
    File genomeReference
    File genomeReferenceIndex
  }

  String base_file_name_tumor = basename(tumor_bam, ".recal.bam")
  String base_file_name_normal = basename(normal_bam, ".recal.bam")
  String ref_fasta_local = basename(refGenome.ref_fasta)
  String genomeReference_local = basename(genomeReference)

  command <<<
    set -eo pipefail

    mv ~{refGenome.ref_fasta} .
    mv ~{refGenome.ref_fasta_index} .
    mv ~{refGenome.ref_dict} .

    mv ~{genomeReference} .
    mv ~{genomeReferenceIndex} .

    gatk --java-options "-Xms16g" Mutect2 \
      -R ~{ref_fasta_local} \
      -I ~{tumor_bam} \
      -I ~{normal_bam} \
      -O preliminary.vcf.gz \
      --germline-resource ~{genomeReference_local} \

    gatk --java-options "-Xms16g" FilterMutectCalls \
      -V preliminary.vcf.gz \
      -O ~{base_file_name_tumor}.mutect2.vcf.gz \
      -R ~{ref_fasta_local} \
      --stats preliminary.vcf.gz.stats \

>>>

  runtime {
    docker: "ghcr.io/getwilds/gatk:4.3.0.0"
    memory: "24 GB"
    cpu: 1
  }

  output {
    File output_vcf = "${base_file_name_tumor}.mutect2.vcf.gz"
    File output_vcf_index = "${base_file_name_tumor}.mutect2.vcf.gz.tbi"
  }
}

# annotate with annovar mutation calling outputs
task annovar {
  input {
    File input_vcf
    String ref_name
    String annovar_protocols
    String annovar_operation
  }
  String base_vcf_name = basename(input_vcf, ".vcf.gz")
  
  command <<<
    set -eo pipefail
  
    perl /annovar/table_annovar.pl ~{input_vcf} /annovar/humandb/ \
      -buildver ~{ref_name} \
      -outfile ~{base_vcf_name} \
      -remove \
      -protocol ~{annovar_protocols} \
      -operation ~{annovar_operation} \
      -nastring . -vcfinput
>>>
  runtime {
    docker : "ghcr.io/getwilds/annovar:${ref_name}"
    cpu: 1
    memory: "2GB"
  }
  output {
    File output_annotated_vcf = "${base_vcf_name}.${ref_name}_multianno.vcf"
    File output_annotated_table = "${base_vcf_name}.${ref_name}_multianno.txt"
  }
}
```


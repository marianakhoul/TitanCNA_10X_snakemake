# *Snakemake workflow for TITAN analysis of 10X Genomics WGS*

## Description
This workflow will run the TITAN copy number analysis for set of tumour-normal pairs, starting from the BAM files aligned using [Long Ranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger) software. The analysis includes haplotype-based copy number prediction and post-processing of results. It will also perform model selection at the end of the workflow to choose the optimal ploidy and clonal cluster solutions.  
Viswanathan SR*, Ha G*, Hoff A*, et al. Structural Alterations Driving Castration-Resistant Prostate Cancer Revealed by Linked-Read Genome Sequencing. *Cell* 174, 433–447.e19 (2018).

## Contact
Gavin Ha  
Fred Hutchinson Cancer Research Center  
contact: <gavinha@gmail.com> or <gha@fredhutch.org>  
Date: August 7, 2018  

## Requirements
### Software packages or libraries
 - R-3.4
   - [TitanCNA](https://github.com/gavinha/TitanCNA) (v1.15.0) or higher
   		- TitanCNA imports: GenomicRanges, GenomeInfoDb, VariantAnnotation, dplyr, data.table, foreach
   - [ichorCNA](https://github.com/broadinstitute/ichorCNA) (v0.1.0) 
   - HMMcopy
   - optparse
   - stringr
   - SNPchip
   - doMC
 - Python 3.4 
   - snakemake-3.12.0
   - PySAM-0.11.2.1
   - PyYAML-3.12
 - [bxtools](https://github.com/walaj/bxtools)


### Scripts used by the workflow
The following scripts are used by this snakemake workflow:
 - [getMoleculeCoverage.R](code/getMoleculeCoverage.R) Normalizing/correcting molecule-level coverage
 - [getPhasedHETSitesFromLLRVCF.R](code/getPhasedHETSitesFromLLRVCF.R) - Extracts phased germline heterozygous SNP sites from the Long Ranger analysis of the normal sample
 - [getTumourAlleleCountsAtHETSites.py](code/getTumourAlleleCountsAtHETSites.py) - Extracts allelic counts from the tumor sample at the germline heterozygous SNP sites
 - [titanCNA_v1.15.0_TenX.R](code/titanCNA_v1.15.0_TenX.R) - Main R script to run TitanCNA
 - [selectSolution.R](code/selectSolution.R) - R script to select optimal solution for each sample
 - [combineTITAN-ichor.R](code/combineTITAN-ichor.R) - R script to merge autosomes and chrX results, plus post-processing steps including adjusting max copy values.

## Tumour-Normal sample list
The list of tumour-normal paired samples should be defined in a YAML file. In particular, the [Long Ranger](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger) (v2.2.2) analysis directory is listed under samples.  See `config/samples.yaml` for an example.  Both fields `samples` and `pairings` must to be provided.  `pairings` key must match the tumour sample while the value must match the normal sample.
```
samples:
  tumor_sample_1:  /path/to/tumor/longranger/dir
  normal_sample_1:  /path/to/normal/longranger/dir


pairings:
  tumor_sample_1:  normal_sample_1
```


## snakefiles
1. [moleculeCoverage.snakefile](moleculeCoverage.snakefile)
2. [getPhasedAlleleCounts.snakefile](getPhasedAlleleCounts.snakefile)
3. [TitanCNA.snakefile](TitanCNA.snakefile)

# Run the analysis

## 1. Invoking the full snakemake workflow for TITAN
This will also run both [moleculeCoverage.snakefile](moleculeCoverage.snakefile) and [getPhasedAlleleCounts.snakefile](getPhasedAlleleCounts.snakefile) which generate the necessary inputs for [TitanCNA.snakefile](TitanCNA.snakefile).
```
# show commands and workflow
snakemake -s TitanCNA.snakefile -np
# run the workflow locally using 5 cores
snakemake -s TitanCNA.snakefile --cores 5
```
Users can launch the snakemake jobs to a cluster.  
An implementation that works with Broad UGER (qsub) is provided.  
Parameters for memory, runtime, and parallel environment can be specified directly in the snakemake files; default values for each rule has already been set in `params` within the [config.yaml](config/config.yaml) and the command below can be used as-is.  
Other cluster parameters can be set directly in [cluster.sh](config/cluster.sh).  
*Note: users will need to adjust these for use with their cluster-specific settings*
```
snakemake -s TitanCNA.snakefile --cluster-sync "qsub -l h_vmem={params.mem},h_rt={params.runtime} {params.pe}" -j 50 --jobscript config/cluster.sh
```


## 2. Invoking individual steps in the workflow
Users can run the snakemake files individually. This can be helpful for testing each step or if you only wish to generate results for a particular step. The snakefiles need to be run in this same order since input files are generated by the previous steps.
  ### a. [moleculeCoverage.snakefile](moleculeCoverage.snakefile)
  i.   Run [bxtools](https://github.com/walaj/bxtools) to compute counts of unique molecules in each window.  
  ii.  Perform GC-content bias correction for barcode counts.  
  iii. Perform ichorCNA analysis to generate initial molecule coverage-based copy number. For male samples, chrX results will be used from this step.  
  ```
  snakemake -s moleculeCoverage.snakefile -np
  snakemake -s moleculeCoverage.snakefile --cores 5
  # OR
  snakemake -s moleculeCoverage.snakefile --cluster-sync "qsub -l h_vmem={params.mem},h_rt={params.runtime} {params.pe}" -j 50 --jobscript config/cluster.sh
  ```
  
  ### b. [getPhasedAlleleCounts.snakefile](getPhasedAlleleCounts.snakefile) 
  i.   Read the Long Ranger output file `*phased_variants.vcf.gz` and extract heterozygous SNP sites (that overlap a SNP database, e.g. [hapmap_3.3.hg38.vcf.gz](https://storage.cloud.google.com/genomics-public-data/resources/broad/hg38/v0/hapmap_3.3.hg38.vcf.gz?_ga=2.110868357.-1633399588.1531762721)). You can find all the hg38 reference files here https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0  
  i.   Extract the allelic read counts from the Long Ranger tumor bam file `phased_possorted_bam.bam` for each chromosome.  
  iii. Cat the allelic read counts from each chromosome file into a single counts file.
  ```
  snakemake -s getPhasedAlleleCounts.snakefile -np
  snakemake -s getPhasedAlleleCounts.snakefile --cores 5
  # OR
  snakemake -s getPhasedAlleleCounts.snakefile --cluster-sync "qsub -l h_vmem={params.mem},h_rt={params.runtime} {params.pe}" -j 50 --jobscript config/cluster.sh
  ``` 
  ### c. [TitanCNA.snakefile](TitanCNA.snakefile)
  i.   Run the [TitanCNA](https://github.com/gavinha/TitanCNA) analysis and generates solutions for different ploidy initializations and each clonal cluster.  
  ii.  Merge results with ichorCNA output generate by [moleculeCoverage.snakefile](moleculeCoverage.snakefile) and post-processes copy number results.  
  iii. Select optimal solution for each samples and copies these to a new folder. The parameters are compiled in a text file.  
  
  
  
  

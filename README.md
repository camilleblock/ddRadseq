# ddRADseq Pipeline


## Introduction
This page is a work in progress!
This repo explains a basic pipeline for ddRADseq analysis. It was developed as part of curriculum for Virginia Tech's Intro to Genomic Data Science course. This pipeline runs in Linux and relies on Stacks, Bowtie2, and samtools.

Contact: Camille Block (camilleblock@vt.edu)

To do (before finalized):
- [ ] Make population map (Stacks doesn't like present format)
- [ ] Run gstacks in stacks
- [ ] Run populations in stacks 

# Install Stacks using EasyBuild
If installation via module load is not available for Stacks use the following EasyBuild code to install the software in your linux environment. Please note that you must select the versions of EasyBuild and Stacks that will work for your pipeline which may alter this code.
```bash
module spider easybuild
module load EasyBuild/4.9.4
eb -S Stacks
eb -r /apps/easybuild/software/tinkercliffs-rome/EasyBuild/4.9.4/easybuild/easyconfigs/s/Stacks/Stacks-2.62-foss-2022a.eb
module load Stacks #testing if install worked
```
# Process radtags 
If your sequences have already been demultiplexed skip this step. Use this step if given raw reads from Illumina sequencing. In order to run this step you must know what restriction enzymes were used in sequencing. 
```bash
process_radtags -p ./in_dir -P -b barcodes.txt -o ./out_dir –renz-2 sphI mluCI –threads 16 -q -r -D -t 120
```
# Download reference genome via ncbi
To start this step ensure that your species has a reference genome on ncbi and find its genome accession number. This code must be edited to fit your target species.
```bash
module spider conda
module load Miniconda3/23.9.0-0
conda create -n ncbi_datasets
#!/bin/bash camilleblock #may be unnecessary depending on environment settings
source ~/.bashrc #may be unnecessary depending on environment settings
conda init –all #may be unnecessary depending on environment settings 
conda activate ncbi_datasets
conda install -c conda-forge ncbi-datasets-cli
datasets
datasets download genome accession GCF_018135715.1 --filename danausgenome.zip
```
# Index genome with Bowtie2
Edit with appropriate files containing reference genome and name of files you are creating.
```bash
module load Bowtie2
bowtie2-build -f GCF_018135715.1_MEX_DaPlex_genomic.fna Danaus
```
# Align genome with Bowtie2
Note: this should be done in globalscratch so make a folder and transfer over files if necessary. This code should be edited with the names of your two files (the forward and reverse read) the name of the joint file you wish to create. This step must be repeated for every genome. 
```bash
bowtie2 -q --phred33 -N 1 -p 8 -x Danaus -1 A005D02.1.fq -2 A005D02.2.fq -S A005D02.sam
```
# Convert from samfile to bamfile using samtools
```bash
SAMtools
samtools view -bS A005D02.sam | samtools sort > A005D02.bam
```
# Run gstacks
I fixed my popmap so I am ready to run gstacks.
```bash
gstacks -I ./bamfiles -O ./gstacks_out -M Dan_info.txt -t 2
```
# Generate statistics using populations
Using populations I will be able to calculate π, FIS, and FST.  Each individual was assigned to their sampling locale in the popmap, with loci present in at least 50% of individuals (--R 0.5), global minor allele frequency of 5% (--min-maf 0.05), and one random SNP per stack. 
```bash
populations -P ./gstacks/ --popmap ./Dan_info.txt --smooth -r 0.55 -min-maf 0.05 -t 8 --write-random-snp
```

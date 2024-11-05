# ddRADseq Pipeline
![image](https://github.com/user-attachments/assets/7719a6bf-3690-40b3-a1bc-030c044408da)
Image from: https://catchenlab.life.illinois.edu/stacks/manual/
# Introduction
This page is a work in progress!
This repo explains a basic pipeline for ddRADseq analysis. It was developed as part of curriculum for Virginia Tech's Intro to Genomic Data Science course. This pipeline runs in Linux and relies on Stacks (version Stacks/2.62-foss-2022a), Bowtie2 (version Bowtie2/2.4.5-GCC-11.3.0), and samtools (version SAMtools/1.16.1-GCC-11.3.0).

Contact: Camille Block (camilleblock@vt.edu)


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
This code turns files into bamfiles which are smaller and easier for programs to handle.
```bash
module load SAMtools
samtools view -bS A005D02.sam | samtools sort > A005D02.bam
```
# Run gstacks
gstacks identifies SNPs within the meta population for each locus and then genotypes each individual at each identified SNP. It also phases the SNPs at each locus into haplotypes.
Before running gstacks make sure your pop map fits the correct format. This map should contain all necessary information about the sampling site and should be saved in .txt file. I recommend making the text file in linux using the nano command. In the example, the numbered groups correspond to levels of socioeconomic status.

Example pop map format:

A005D02	A0005	Toronto_Fifth

A025D01	A025	Toronto_Second

A025D06	A025	Toronto_Second

A048D01	A048	Toronto_Third

A079D02	A079	Toronto_Third

M080D01	M080	Toronto_Second

G02D01	G02	Guelph_Fourth

G15D01	G15	Guelph_Fifth

```bash
gstacks -I ./bamfiles -O ./gstacks_out -M Dan_info.txt -t 2
```
# Generate statistics using populations
Using populations I will be able to calculate π, FIS, and FST.  Each individual was assigned to their sampling locale in the popmap, with loci present in at least 50% of individuals (--R 0.5), global minor allele frequency of 5% (--min-maf 0.05), and one random SNP per stack. Populations also exports SNP data into standard output formats (.tsv). Before running make sure to make a gstacks folder and add gstacks outputs to this folder.
```bash
populations -P ./gstacks/ --popmap ./Dan_info.txt --smooth -r 0.55 --min-maf 0.05 -t 8 --write-random-snp
```
# References
https://catchenlab.life.illinois.edu/stacks/

https://www.htslib.org/doc/samtools.html

https://bowtie-bio.sourceforge.net/bowtie2/index.shtml

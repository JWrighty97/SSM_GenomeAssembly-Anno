# SSM_GenomeAssembly-Anno
Genome assembly and annotation of the Chromosome-level Scaly sided merganser genome with further analysis including demographic history and syntenic analysis
 
Please not most software was already installed on a .yaml (lrgenomics) file provided by coauthor Dr. H. De Weerd from Edinburgh Genomics. 

# Raw data QC & trimming

``` conda activate lrgenomics

Nanoplot -t 16 --fastq nano_cat.fastq.gz --loglength -o Nanopore_nanoplot --plots dot

nanoQC -o Nanopore_nanoQC nano_cat.fastq.gz

#the first 27 and last 38 bases of each subset needed trimming, mean quality of the data was Q17.

gunzip -c nano_cat.fastq.gz | chopper --threads 16 -q 17 -l 500 --headcrop 27 --tailcrop 38 | gzip nano_cat_filt.fastq.gz

#reQC the data

nanoQC -o Nano_trimmed_QC nano_cat_filt.fastq.gz

```

```
trimmomatic PE -threads 30 merged_R1.fq.gz merged_R2.fq.gz \
R1_paired.fq.gz R1_unpaired.fq.gz \
R2_paired.fq.gz R2_unpaired.fq.gz \
LERADING:3 TRAILING:3 SLIDINGWINDOW:4:20 MINLEN:50

fastqc -t 30 R1_paired.fq.gz R2_paired.fq.gz

```

# Genome assembly

flye --nano-hq nano_cat_filt.fastq.gz --out-dir Flye_assem --genome-size 1.2G --asm-coverage 60 --threads 30
#Genome size estimated by GOAT, coverage 60 used to reduce computation power as read file was so large

# Polishing 

medaka_consensus -i nano_cat_filt.fastq.gz -d flye_assem.fasta -o medaka -t 30

nextPolish run.cfg #cfg attached as example

```







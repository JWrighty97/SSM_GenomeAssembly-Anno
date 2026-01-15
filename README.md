# SSM_GenomeAssembly-Anno
Genome assembly and annotation of the Chromosome-level Scaly sided merganser genome with further analysis including demographic history and syntenic analysis
 
Please not most software was already installed on a .yaml (lrgenomics) file provided by coauthor Dr. H. De Weerd from Edinburgh Genomics. 

# Raw data QC & Trimming

``` conda activate lrgenomics

Nanoplot -t 16 --fastq nano_cat.fastq.gz --loglength -o Nanopore_nanoplot --plots dot

nanoQC -o Nanopore_nanoQC nano_cat.fastq.gz

#the first 27 and last 38 bases of each subset needed trimming, mean quality of the data was Q17.

gunzip -c nano_cat.fastq.gz | chopper --threads 16 -q 17 -l 500 --headcrop 27 --tailcrop 38 | gzip nano_cat_filt.fastq.gz

#reQC the data

nanoQC -o Nano_trimmed_QC nano_cat_filt.fastq.gz

```



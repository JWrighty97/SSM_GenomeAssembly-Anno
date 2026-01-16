# SSM_GenomeAssembly-Anno
Genome assembly and annotation of the Chromosome-level Scaly sided merganser genome with further analysis including demographic history and syntenic analysis
 
Please note most software was already installed on a .yaml (lrgenomics) file provided by coauthor Dr. H. De Weerd from Edinburgh Genomics. 

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

```

flye --nano-hq nano_cat_filt.fastq.gz --out-dir Flye_assem --genome-size 1.2G --asm-coverage 60 --threads 30
#Genome size estimated by GOAT, coverage 60 used to reduce computation power as read file was so large

```

# BUSCO 
This was ran at each step of the genome assembly to ensure quality/completeness remained high

```

busco -i assembly.fasta -o Flye_busco -m genome --lineage aves_odb12 -c 30

```

# Polishing 

```

medaka_consensus -i nano_cat_filt.fastq.gz -d flye_assem.fasta -o medaka -t 30
busco -i consensus.fasta -o Med_buco --lineage aves_odb12 -t 30

nextPolish run.cfg #cfg attached as example
busco -i assembly.nexpolish.fasta -o NP_busco --lineage aves_odb12 -t 30

```
# Hi-C assembly

```

mkdir Flye_improvements

mv SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_1.fq SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_2.fq ./Flye_improvements
mv assembly.nextpolish.fasta ./Flye_improvements

#Index genome & align HiC reads
bwa index assembly.nextpolish.fasta

bwa mem -A1 -B4 -E50 -L0 assembly.nextpolish.fasta SM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_1.fq 2>> mate1.log | samtools view -Shb - > mate_R1.bam

bwa mem -A1 -B4 -E50 -L0 assembly.nextpolish.fasta SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_2.fq 2>> mate2.log | samtools view -Shb - > mate_R2.bam

#Digest genome using enzymes used during hi-c process
hicFindRestSite --fasta assembly.nextpolish.fasta --searchPattern .GATC G.ANTC --outFile GenomeDigest.bed

#Build Hic matrix to QC

hicBuildMatrix --samFiles mate_R1.bam mate_R2.bam --restrictionSequence GATC GAATC GACTC GAGTC GATTC --danglingSequence GATC AAT ACT ATT --restrictionCutFile GenomeDigest.bed --threads 30 --outBam hiCExplorer.bam -o hic_matrix.h5 --QCfolder ./HiCExplorerQC

#After QC move onto YAHS

mkdir yahsresults
cd yahsresults

ln -s ../assembly.nextpolish.fasta ./
samtools faidx assembly.nextpolish.fasta

yahs assembly.nextpolish.fasta ../HiCExplorer.bam

#Another BUSCO
busco -i yahs.out.scaffolds_final.fa -o busco_yahs -m gneome -l aves_odb12 -f -c 30

```










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

# Tiara Decontamination 

```

cd ~/
mkdir tiara
cd tiara
ln -s ~/Flye_improvements/yahsresults/yahs.out.scaffolds_final.fa 

tiara -i yahsresults/yahs.out.scaffolds_final.fa  -o primary.tiara -t 30 --pr --tf all -m 1000

mv eukarya_yahs.out.scaffolds_final.fa primary_yahs_tiara.fa


```

# TreeVal

```
# Treeval set up

#Set up directory structure
cd ~/
mkdir treeval-resources
mkdir -p treeval-resources/gene_alignment_prep/scripts/
cp bin/treeval-dataprep/* treeval-resources/gene_alignment_prep/scripts/
mkdir -p treeval-resources/gene_alignment_prep/raw_fasta/
mkdir -p treeval-resources/gene_alignment_data/birds/csv_data/
mkdir -p treeval-resources/synteny/birds/

#Place reference genomes used for synteny in /synteny/birds directory 
#Place assembly in raw_fasta using the decontaminated assembly

cd ~/treeval/treeval-resources/gene_alignment_prep/raw_fasta
ln -s ~/tiara/primary/primary_yahs_tiara.fa MergusSquamatus-primary.cdna.fa

#prep data

python3 ../scripts/GA_data_prep.py MergusSquamatus-primary.cdna.fa ncbi 10
#Which creates a directory that must be moved to final location

mv MergusSquamatus/ ../../gene_alignment_data/birds/

#Generate CSV

cd ~/treeval/treeval-resources/gene_alignment_prep/

python3 scripts/GA_csv_gen.py ~/treeval/treeval-resources/gene_alignment_data/

# Hi-C prepatation 

cd ~/treeval
mkdir hic_data
cd hic_data #move raw hic files here

samtools import -@ 20 \
    -r ID:SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2 \
    -r CN:Qiagen_EpiTect \
    -r PU:SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2 \
    -r SM:Q31 \
    SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_1.fq.gz \
    SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2_2.fq.gz \
    -o SSM_HiC_EKDL240007522-1A_HHTKLDSXC_L2.cram
samtools index *.cram


#Nanopore data prep - turn fastq.gz into fasta

cd ~/treeval
mkdir novo_data
cd nano_data #Move filtered nanopore data here

seqtk seq -a nano_cat_filt.fastq.gz > nano.fasta
pigz -p100 -c nano.fasta > nano.fasta.gz


#Run Treeval
cd ~/
nextflow run treeval/main.nf --input treeval/treeval_primary.yaml --outdir ~/treeval/primary_out_treeval -profile singularity 

```

# Manual Curation

```
#Open primary_1_normal.pretext in PreText view & curate
#save agp & state before closing
#make new dir post_curation and move agp here

mkdir post_curation
ln -s ~/tiara/primary/primary_yahs_tiara.fa MergusSquamatus-primary.cdna.fa






# Mitogenome assembly and annotation

```
#Iteration 1 of mitofinder was used to find the most appropriate contig

mitofinder -j scaly_mitogenome -1 R1_paired.fq.gz -2 R2_paired.fq.gz -r Merg_ref_mito.gb -o 2 --circular-size 45 --circular-offset 200
#circularisation used at default parameters -o 2 is genetic code for vertebrates

#mitofinder iteration 2 used best derived contig against reference mitogenome to refine
mitofinder -j scaly_mitogenome -r Merg_ref_mito.gb -a Scaly_mitogenome_mtDNA_contig_1.fasta -o 2 --circular-size 45 --circular-offset 200

```
# Repeat annotation















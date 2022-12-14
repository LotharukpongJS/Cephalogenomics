
# _Epigonichthys_ genome assembly

## Pre-assembly analysis

### Illumina reads

```bash
../jellyfish/bin/jellyfish count -C -m 21 -s 1000M -t 10 EPI_*.fastq -o EPI_both.jf
../jellyfish/bin/jellyfish histo -t 10 EPI_both.jf > EPI_both.histo
```

The resulting distribution was inputted into GenomeScope with default options of 

```
p = 2
k = 21
```
and
```
Max k-mer coverage = -1
Average k-mer coverage for polyploid genome = -1
```
http://qb.cshl.edu/genomescope/genomescope2.0/analysis.php?code=Y5BCeyMIUGNPBt1q07DK

### Nanopore reads

```bash
NanoPlot \
-t 8 \
-o ONT_stats \
-p EPI \
-c Pastel1 \
--N50 \
--title "EPI" \
--fastq EPI_all2_guppy4.fastq.gz
```

```bash
NanoPlot \
-t 8 \
-o ONT_stats \
-p EPI_log \
--loglength \
--N50 \
--title "EPI" \
--fastq EPI_all2_guppy4.fastq.gz
```

## Genome assembly

MaSuRCA

### MaSuRCA config file

```bash
DATA
#Illumina paired end reads supplied as <two-character prefix> <fragment mean> <fragment stdev> <forward_reads> <reverse_reads>
#if single-end, do not specify <reverse_reads>
#MUST HAVE Illumina paired end reads to use MaSuRCA
PE= pe 330 100  /hps/nobackup/research/marioni/sodai/EPI_R1.fastq.gz  /hps/nobackup/research/marioni/sodai/EPI_R2.fastq.gz
#Illumina mate pair reads supplied as <two-character prefix> <fragment mean> <fragment stdev> <forward_reads> <reverse_reads>
#JUMP= sh 3600 200  /FULL_PATH/short_1.fastq  /FULL_PATH/short_2.fastq
#pacbio OR nanopore reads must be in a single fasta or fastq file with absolute path, can be gzipped
#if you have both types of reads supply them both as NANOPORE type
#PACBIO=/FULL_PATH/pacbio.fa
NANOPORE=/hps/nobackup/research/marioni/sodai/EPI_all2_guppy4.fastq.gz
#Other reads (Sanger, 454, etc) one frg file, concatenate your frg files into one if you have many
#OTHER=/FULL_PATH/file.frg
#synteny-assisted assembly, concatenate all reference genomes into one reference.fa; works for Illumina-only data
#REFERENCE=/FULL_PATH/nanopore.fa
END

PARAMETERS
#PLEASE READ all comments to essential parameters below, and set the parameters according to your project
#set this to 1 if your Illumina jumping library reads are shorter than 100bp
EXTEND_JUMP_READS=0
#this is k-mer size for deBruijn graph values between 25 and 127 are supported, auto will compute the optimal size based on the read data and GC content
GRAPH_KMER_SIZE = auto
#set this to 1 for all Illumina-only assemblies
#set this to 0 if you have more than 15x coverage by long reads (Pacbio or Nanopore) or any other long reads/mate pairs (Illumina MP, Sanger, 454, etc)
USE_LINKING_MATES = 0
#specifies whether to run the assembly on the grid
USE_GRID=0
#specifies grid engine to use SGE or SLURM
#GRID_ENGINE=SGE
#specifies queue (for SGE) or partition (for SLURM) to use when running on the grid MANDATORY
#GRID_QUEUE=all.q
#batch size in the amount of long read sequence for each batch on the grid
#GRID_BATCH_SIZE=500000000
#use at most this much coverage by the longest Pacbio or Nanopore reads, discard the rest of the reads
#can increase this to 30 or 35 if your reads are short (N50<7000bp)
LHE_COVERAGE=60
#set to 0 (default) to do two passes of mega-reads for slower, but higher quality assembly, otherwise set to 1
MEGA_READS_ONE_PASS=0
#this parameter is useful if you have too many Illumina jumping library mates. Typically set it to 60 for bacteria and 300 for the other organisms
#LIMIT_JUMP_COVERAGE = 300
#these are the additional parameters to Celera Assembler.  do not worry about performance, number or processors or batch sizes -- these are computed automatically.
#CABOG ASSEMBLY ONLY: set cgwErrorRate=0.25 for bacteria and 0.1<=cgwErrorRate<=0.15 for other organisms.
#CA_PARAMETERS =  cgwErrorRate=0.15
#CABOG ASSEMBLY ONLY: whether to attempt to close gaps in scaffolds with Illumina  or long read data
#CLOSE_GAPS=1
#number of cpus to use, set this to the number of CPUs/threads per node you will be using
NUM_THREADS = 32
#this is mandatory jellyfish hash size -- a safe value is estimated_genome_size*20
JF_SIZE = 10000000000 #500mb*20
#ILLUMINA ONLY. Set this to 1 to use SOAPdenovo contigging/scaffolding module.
#Assembly will be worse but will run faster. Useful for very large (>=8Gbp) genomes from Illumina-only data
SOAP_ASSEMBLY=0
#If you are doing Hybrid Illumina paired end + Nanopore/PacBio assembly ONLY (no Illumina mate pairs or OTHER frg files).
#Set this to 1 to use Flye assembler for final assembly of corrected mega-reads.
#A lot faster than CABOG, AND QUALITY IS THE SAME OR BETTER.
#Works well even when MEGA_READS_ONE_PASS is set to 1.
#DO NOT use if you have less than 15x coverage by long reads.
FLYE_ASSEMBLY=1
END
```

Where `/hps/nobackup/research/marioni/sodai/EPI_all2_guppy4.fastq.gz` is the full path to the long reads and `/hps/nobackup/research/marioni/sodai/EPI_R1.fastq.gz` and `/hps/nobackup/research/marioni/sodai/EPI_R2.fastq.gz` are the full paths to the short reads (forward and reverse).

### MaSuRCA assembly (v3.4.2)

```bash
./masurca sr_config_epi_lhe60.txt
```

Where `sr_config_epi_lhe60.txt` is the config file for _Epigonichthys_ with option `LHE_COVERAGE=60`. This generates a configuration shell script `assembly.sh`, which is run to assemble the data.

```bash
./assemble.sh
```

## Post-assembly processing

### Polishing using POLCA

POLCA is from the MaSuRCA v3.4.2 toolkit.

```bash
./polca.sh \
-a /hps/nobackup/research/marioni/sodai/EPI_masurca_LHE60_rerun/flye/assembly.fasta \
-r '../../EPI_R1.fastq.gz ../../EPI_R2.fastq.gz' \
-t 16 \
-m 1G
```

Where `EPI_R1.fastq.gz` and `EPI_R2.fastq.gz` are the forward (R1) and reverse (R2) short reads. `/hps/nobackup/research/marioni/sodai/EPI_masurca_LHE60_rerun/flye/assembly.fasta` is the full path to the assembly.

### Haplotig purging

#### Mapping short reads using BWA MEM (BWA v0.7.17)

Create an index

```bash
bwa index asm.fasta

```

Where `asm.fasta` is the polished assembly.

Map short reads

```bash
bwa mem -t 14 asm.fasta EPI_R1.fastq.gz EPI_R2.fastq.gz > out.sam
```

Where ```EPI_R1.fastq.gz``` and ```EPI_R2.fastq.gz``` are forward (R1) and reverse (R2) reads.

Convert output from .sam to .bam

```bash
samtools view -S -b out.sam > out.bam
```
#### purge_dups v1.2.5

Calculate read depth histogram

```bash
./purge_dups/src/ngscstat out.bam
```

Where `out.bam` is the output from the alignment step.

Calculate base-level read depth

```bash
./purge_dups/bin/calcuts TX.stat > cutoffs 2>calcults.log
```

The custom cutoffs are `5 84 84 85 85 225`

Split an assembly and do a self-self alignment

```bash
./purge_dups/bin/split_fa asm.fasta > asm.split
```

```bash
minimap2 -xasm20 -DP asm.split asm.split | gzip -c - > asm.split.self.paf.gz
```

Where `asm.split` is the split assembly.

Purge haplotigs and overlaps

```bash
./purge_dups/bin/purge_dups -2 -T cutoffs -c TX.base.cov asm.split.self.paf.gz > dups.bed 2> purge_dups.log
```

Where `cutoffs` is a file containing the manually calculated cutoffs.

Get purged primary and haplotig sequences

```bash
./purge_dups/bin/get_seqs dups.bed asm.fasta
```

Where `.bed` file `dups.bed` contains the coordinates for purging. Notice, `-e` was not included.

### Repeat-masking

#### Repeat-masking using RepeatModeler and RepeatMasker (from TETools 1.3)

Build database

```bash
singularity exec docker://dfam/tetools:latest BuildDatabase \
-name EPI \
purged.fa
```

Where `purged.fa` is the purged assembly.

Model repeats using RepeatModeler

```bash
singularity exec docker://dfam/tetools:latest RepeatModeler \
-database EPI \
-pa 6 \
-LTRStruct
```

Mask repeats using RepeatMasker

```bash
singularity exec docker://dfam/tetools:latest RepeatMasker \
-lib EPI-families.fa \
purged.fa \
-pa 6 \
-xsmall
```


### Repeat-masking

#### Repeat-masking using RepeatModeler and RepeatMasker (from TETools 1.3)

Build database

```bash
singularity exec docker://dfam/tetools:latest BuildDatabase \
-name ASY \
purged.fa
```

Where `purged.fa` is the purged assembly.

Model repeats using RepeatModeler

```bash
singularity exec docker://dfam/tetools:latest RepeatModeler \
-database ASY \
-pa 6 \
-LTRStruct
```

Mask repeats using RepeatMasker

```bash
singularity exec docker://dfam/tetools:latest RepeatMasker \
-lib ASY-families.fa \
purged.fa \
-pa 6 \
-xsmall
```

### RNA-scaffolding

Hybrid approach using both soft- and hard-masked genomes.

#### Map RNA-seq reads using hisat2

Hard-mask the soft-masked assembly

```bash
sed '/^[^>]/s/[a-z]/N/g'<EPI_2_RM.fa >EPI_2_RM_hard.fa
```

Where `EPI_2_RM.fa` is the repeat-masked (and polished and haplotig-purged) genome (renamed from `purged.fa.masked`).

Index hard-masked genome

```bash
hisat2-build \
EPI_2_RM_hard.fa \
EPI_2_RM_hard
```

Where `EPI_2_RM_hard.fa` is the hard-masked genome.

Align RNA-seq reads

```bash
hisat2 \
-x EPI_2_RM_hard \
-1 ../../RNA_preprocessing/EPI_RNA_R1_trimmed.fq.gz \
-2 ../../RNA_preprocessing/EPI_RNA_R2_trimmed.fq.gz \
-k 3 \
-p 10 \
--pen-noncansplice 1000000 \
-S input.sam
```

Where `EPI_RNA_R1_trimmed.fq.gz` and `EPI_RNA_R2_trimmed.fq.gz` are the forward (R1) and reverse (R2) trimmed RNA-seq reads, and `EPI_2_RM_hard` is the hard-masked index. These were trimmed using trimmomatic, i.e.

```bash
trimmomatic PE \
-threads 10 \
-trimlog trim_EPI.log \
../EPI_RNA_R1_QC.fastq \
../EPI_RNA_R2_QC.fastq \
EPI_RNA_R1_trimmed.fq.gz \
EPI_RNA_R1_unpaired.fq.gz \
EPI_RNA_R2_trimmed.fq.gz \
EPI_RNA_R2_unpaired.fq.gz \
ILLUMINACLIP:TruSeq2-PE.fa:4:30:10 \
SLIDINGWINDOW:5:15
```

Where `EPI_RNA_R1_QC.fastq` and `EPI_RNA_R2_QC.fastq` are the raw reads.

#### P_RNA_scaffolder

Run P_RNA_scaffolder on the soft-masked genome

```bash
./P_RNA_scaffolder/P_RNA_scaffolder_edit.sh \
-d ../../P_RNA_scaffolder \
-i input.sam \
-j EPI_2_RM.fa \
-F ../../RNA_preprocessing/EPI_RNA_R1_trimmed.fq.gz \
-R ../../RNA_preprocessing/EPI_RNA_R2_trimmed.fq.gz \
-t 10 \
-o scaffold
```

Where input.sam is the mapping file from the hard-masked genome, `EPI_2_RM.fa` is the soft-masked assembly, and `EPI_RNA_R1_trimmed.fq.gz` and `EPI_RNA_R2_trimmed.fq.gz` are the forward (R1) and reverse (R2) trimmed RNA-seq reads. The `P_RNA_scaffolder_edit.sh` has been edited, i.e. `-masked=soft` was added for BLAT pairwise aligner.


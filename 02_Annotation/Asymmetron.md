# _Asymmetron_ genome annotation

## BRAKER ET mode

### Align RNA-seq reads using Hisat2

Index the draft genome

```bash
hisat2-build \
ASY_3_scaf.fa \
ASY_3_scaf
```
Where `ASY_3_scaf.fa` is the scaffolded assembly (repeat-masked, haplotig-purged and polished).

Map the RNA-seq reads

```bash
hisat2 \
-x ASY_3_scaf \
-1 ../../RNA_preprocessing/ASY_RNA_R1_trimmed.fq.gz \
-2 ../../RNA_preprocessing/ASY_RNA_R2_trimmed.fq.gz \
-S input.sam
```

Where `ASY_RNA_R1_trimmed.fq.gz` and `ASY_RNA_R2_trimmed.fq.gz` are the forward (R1) and reverse (R2) trimmed RNA-seq reads. For trimming protocol, see the [RNA-scaffolding step](https://github.com/LotharukpongJS/Cephalogenomics/blob/main/01_Assembly/Asymmetron.md#rna-scaffolding).

Convert `.sam` to `.bam`

```bash
samtools view -bS input.sam > input.bam
```
Sort

```bash
samtools sort -n -@ 4 -m 2G input.bam -o input.sorted.bam
```

### BRAKER ET mode (BRAKER v2.1.6)

```bash
braker.pl \
--species=ASY \
--genome=ASY_3_scaf.fa \
--bam=input.sorted.bam \
--softmasking \
--cores=20
```

Where `input.sorted.bam` is the RNA-seq read alignment file and `ASY_3_scaf.fa` is the scaffolded assembly.

## Protein sequence extraction

### Get CDS using getAnnoFasta.pl (BRAKER v2.1.6)

```bash
getAnnoFasta.pl ../braker/braker.gtf --seqfile ../ASY_3_scaf.fa
```

Where `ASY_3_scaf.fa` is the scaffolded genome and `braker.gtf` is the BRAKER annotation file.

### Use transeq from EMBOSS to translate CDS

```bash
transeq ../braker/braker.codingseq ASY.faa
```

Where `braker.codingseq` is the output of the previous step.

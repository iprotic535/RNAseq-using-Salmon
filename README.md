# RNA-seq Quantification Workflow Using Salmon

This repository documents a simple, reproducible RNA-seq workflow for transcript-level expression quantification using:

**SRA Toolkit → fastp → Salmon**

The workflow can be used for both **single-end (SE)** and **paired-end (PE)** RNA-seq data.

The general analysis is:

```text
SRA accession / FASTQ
        ↓
SRA Toolkit
        ↓
Raw FASTQ
        ↓
fastp
        ↓
Trimmed FASTQ
        ↓
Salmon transcriptome index
        ↓
Salmon quantification
        ↓
quant.sf
```

---

# 1. Required software

Install:

- SRA Toolkit
- fastp
- Salmon

A Conda/Mamba environment can be created with:

```bash
mamba create -n rnaseq \
    -c conda-forge \
    -c bioconda \
    sra-tools \
    fastp \
    salmon
```

Activate the environment:

```bash
conda activate rnaseq
```

Check versions:

```bash
prefetch --version
fasterq-dump --version
fastp --version
salmon --version
```

For reproducibility, record the versions used in the final analysis.

---

# 2. Recommended project structure

```text
RNAseq_project/
├── raw/
│   └── sra/
├── trimmed/
├── ref/
│   ├── transcripts.fa
│   └── salmon_index/
└── output/
    ├── salmon/
    └── tmp/
```

Create the directories:

```bash
mkdir -p raw/sra
mkdir -p trimmed
mkdir -p ref
mkdir -p output/salmon
mkdir -p output/tmp/fasterq
```

---

# 3. Reference transcriptome

Salmon quantifies RNA-seq reads against a transcriptome reference.

Place the transcript FASTA in:

```text
ref/transcripts.fa
```

For a species-specific project, rename or link the downloaded cDNA/transcript FASTA:

```bash
ln -s /path/to/species_transcripts.fa ref/transcripts.fa
```

The transcriptome should correspond to the genome assembly and annotation release used in the project.

Record:

```text
Species:
Genome assembly:
Annotation release:
Transcriptome source:
Transcriptome filename:
Download date:
```

---

# 4. Download RNA-seq reads from SRA

Suppose the run accession is:

```text
SRR6305029
```

Set a shell variable:

```bash
SAMPLE="SRR6305029"
```

Download the SRA run:

```bash
prefetch "$SAMPLE" \
    --output-directory raw/sra
```

The downloaded run is usually stored as:

```text
raw/sra/SRR6305029/
```

---

# 5. Convert SRA to FASTQ

The command differs depending on whether the sequencing layout is single-end or paired-end.

Before analysis, check the experiment metadata and determine whether the run is:

```text
SINGLE
```

or:

```text
PAIRED
```

Do not infer layout only from the accession number.

---

# 6. Single-end RNA-seq

## 6.1 Convert SRA to FASTQ

For single-end reads:

```bash
fasterq-dump raw/sra/"$SAMPLE" \
    --threads 16 \
    --temp output/tmp/fasterq \
    --outdir raw
```

Expected output:

```text
raw/SRR6305029.fastq
```

---

## 6.2 Trim and filter with fastp

Run:

```bash
fastp \
    -i raw/"${SAMPLE}.fastq" \
    -o trimmed/"${SAMPLE}.trimmed.fastq.gz" \
    --qualified_quality_phred 20 \
    --length_required 30 \
    --thread 16 \
    --html trimmed/"${SAMPLE}_fastp.html" \
    --json trimmed/"${SAMPLE}_fastp.json"
```

Outputs:

```text
trimmed/SRR6305029.trimmed.fastq.gz
trimmed/SRR6305029_fastp.html
trimmed/SRR6305029_fastp.json
```

### Parameters

```text
--qualified_quality_phred 20
```

uses a Phred quality threshold of 20.

```text
--length_required 30
```

removes reads shorter than 30 nt after filtering.

```text
--thread 16
```

uses 16 CPU threads.

Inspect the fastp HTML report before continuing.

Important features include:

- number of input reads
- number of reads passing filters
- per-base quality
- adapter content
- read-length distribution
- GC distribution
- overrepresented sequences

---

# 7. Paired-end RNA-seq

For paired-end data, both mates must be retained.

## 7.1 Convert SRA to paired FASTQ

Run:

```bash
fasterq-dump raw/sra/"$SAMPLE" \
    --split-files \
    --threads 16 \
    --temp output/tmp/fasterq \
    --outdir raw
```

Expected output:

```text
raw/SRRXXXXXXX_1.fastq
raw/SRRXXXXXXX_2.fastq
```

The files correspond to:

```text
_1 = read 1
_2 = read 2
```

---

## 7.2 Trim paired-end reads

Run:

```bash
fastp \
    -i raw/"${SAMPLE}_1.fastq" \
    -I raw/"${SAMPLE}_2.fastq" \
    -o trimmed/"${SAMPLE}_R1.trimmed.fastq.gz" \
    -O trimmed/"${SAMPLE}_R2.trimmed.fastq.gz" \
    --qualified_quality_phred 20 \
    --length_required 30 \
    --thread 16 \
    --html trimmed/"${SAMPLE}_fastp.html" \
    --json trimmed/"${SAMPLE}_fastp.json"
```

Outputs:

```text
trimmed/SRRXXXXXXX_R1.trimmed.fastq.gz
trimmed/SRRXXXXXXX_R2.trimmed.fastq.gz
trimmed/SRRXXXXXXX_fastp.html
trimmed/SRRXXXXXXX_fastp.json
```

---

# 8. Build the Salmon transcriptome index

The Salmon index is built from the transcript FASTA.

Run:

```bash
salmon index \
    -t ref/transcripts.fa \
    -i ref/salmon_index \
    -p 16
```

This creates:

```text
ref/salmon_index/
```

The index only needs to be rebuilt when the reference transcriptome changes.

Examples include:

```text
different species
different genome assembly
different transcript annotation release
```

---

# 9. Quantify single-end reads with Salmon

For single-end RNA-seq:

```bash
mkdir -p output/salmon/"$SAMPLE"
```

Run:

```bash
salmon quant \
    -i ref/salmon_index \
    -l A \
    -r trimmed/"${SAMPLE}.trimmed.fastq.gz" \
    --validateMappings \
    --seqBias \
    --gcBias \
    -p 16 \
    --numBootstraps 100 \
    -o output/salmon/"$SAMPLE"
```

The important single-end argument is:

```text
-r
```

which supplies the single FASTQ file.

---

# 10. Quantify paired-end reads with Salmon

For paired-end RNA-seq:

```bash
mkdir -p output/salmon/"$SAMPLE"
```

Run:

```bash
salmon quant \
    -i ref/salmon_index \
    -l A \
    -1 trimmed/"${SAMPLE}_R1.trimmed.fastq.gz" \
    -2 trimmed/"${SAMPLE}_R2.trimmed.fastq.gz" \
    --validateMappings \
    --seqBias \
    --gcBias \
    --numBootstraps 100 \
    -p 16 \
    -o output/salmon/"$SAMPLE"
```

The important paired-end arguments are:

```text
-1
```

for mate 1 and:

```text
-2
```

for mate 2.

The central distinction is therefore:

```text
Single-end:
-r reads.fastq.gz
```

versus:

```text
Paired-end:
-1 reads_R1.fastq.gz
-2 reads_R2.fastq.gz
```

---

# 11. Salmon options used in this workflow

## `-i`

```bash
-i ref/salmon_index
```

Specifies the Salmon transcriptome index.

---

## `-l A`

```bash
-l A
```

Requests automatic library-type inference.

Salmon determines the most compatible library configuration from the reads.

If the original experiment clearly reports strandedness, record that information and verify that the inferred library type is reasonable.

---

## `-r`

```bash
-r reads.fastq.gz
```

Specifies a single-end FASTQ file.

---

## `-1` and `-2`

```bash
-1 reads_R1.fastq.gz
-2 reads_R2.fastq.gz
```

Specify paired-end mates.

---

## `--validateMappings`

```bash
--validateMappings
```

Uses Salmon's mapping-validation procedure during transcript quantification.

This improves the specificity of fragment-to-transcript assignment.

---

## `--seqBias`

```bash
--seqBias
```

Models sequence-specific technical bias associated with RNA-seq library preparation.

---

## `--gcBias`

```bash
--gcBias
```

Models fragment-level GC-content bias.

---

## `-p`

```bash
-p 16
```

Specifies the number of CPU threads.

---

# 12. How Salmon handles reads compatible with multiple transcripts

This is important for experiments involving duplicate genes.

Suppose a read is compatible with two similar transcripts:

```text
RNA read
   |
   +----> Transcript A
   |
   +----> Transcript B
```

Salmon does not require all such ambiguous reads to be discarded.

Instead, it retains transcript-compatibility information and incorporates ambiguous fragments into its abundance-estimation model.

A simplified example is:

```text
60 reads mainly support transcript A
20 reads mainly support transcript B
20 reads are compatible with both
```

Salmon uses all of the available evidence when estimating transcript abundance.

This is especially useful for genes with highly similar transcript sequences, including paralogs.

---

# 13. Is Salmon paralog-aware?

Not in an evolutionary sense.

Salmon does not know whether a gene is:

```text
Singleton
Young_duplicate
Old_duplicate
```

Salmon only uses sequence information and fragment-to-transcript compatibility.

The duplication category must therefore be added downstream.

Conceptually:

```text
RNA-seq
   ↓
Salmon
   ↓
Transcript abundance
   ↓
Gene-level abundance
   ↓
Gene annotation
   ↓
Singleton / Young duplicate / Old duplicate
```

---

# 14. Limitation for very young paralogs

Very young duplicates can have almost identical transcript sequences.

For example:

```text
Paralog A:
ATGCTGACTGACCTGATC

Paralog B:
ATGCTGACTGACCTGATC
```

If an RNA-seq read comes entirely from a region that is identical between the two genes, the sequence does not contain enough information to determine its true origin.

Salmon can model this ambiguity, but it cannot create distinguishing sequence information.

This is particularly important for:

- very young duplicates
- recent tandem duplicates
- highly similar paralogs
- short reads
- genes with very few unique sequence regions

Paired-end reads can sometimes improve discrimination because both mates provide sequence information.

---

# 15. Salmon output

The primary output is:

```text
output/salmon/<sample>/quant.sf
```

For example:

```text
output/salmon/SRR6305029/quant.sf
```

The file contains:

```text
Name
Length
EffectiveLength
TPM
NumReads
```

Example:

```text
Name        Length  EffectiveLength  TPM     NumReads
TX001       1500    1300.5           12.4    318.7
TX002       850     651.2             4.2     54.9
```

---

## `Name`

Transcript identifier.

---

## `Length`

Annotated transcript length.

---

## `EffectiveLength`

Transcript length adjusted by Salmon's abundance model.

---

## `TPM`

Transcripts Per Million.

TPM is useful for:

- visualization
- relative expression comparison
- descriptive abundance summaries

---

## `NumReads`

Estimated read/fragment contribution for each transcript.

These values may be fractional because Salmon can probabilistically allocate ambiguous evidence.

---

# 16. Check Salmon results

Do not rely only on the presence of `quant.sf`.

Inspect the Salmon output directory:

```text
output/salmon/<sample>/
```

Check:

- mapping rate
- total processed reads/fragments
- inferred library type
- warnings in Salmon logs
- consistency among biological replicates

A low mapping rate may indicate:

```text
wrong reference transcriptome
incorrect species
poor read quality
contamination
incomplete annotation
very short reads
incorrect sequencing layout
```

---

# 17. Multiple samples

For multiple runs, repeat the same workflow for each accession.

Example:

```bash
SAMPLES=(
    SRR000001
    SRR000002
    SRR000003
)
```

Then each sample should produce an independent directory:

```text
output/salmon/SRR000001/
output/salmon/SRR000002/
output/salmon/SRR000003/
```

The important final file from each sample is:

```text
quant.sf
```

---

# 18. Example complete single-end workflow

Set the sample:

```bash
SAMPLE="SRR6305029"
```

Download:

```bash
prefetch "$SAMPLE" \
    --output-directory raw/sra
```

Convert:

```bash
fasterq-dump raw/sra/"$SAMPLE" \
    --threads 16 \
    --temp output/tmp/fasterq \
    --outdir raw
```

Trim:

```bash
fastp \
    -i raw/"${SAMPLE}.fastq" \
    -o trimmed/"${SAMPLE}.trimmed.fastq.gz" \
    --qualified_quality_phred 20 \
    --length_required 30 \
    --thread 16 \
    --html trimmed/"${SAMPLE}_fastp.html" \
    --json trimmed/"${SAMPLE}_fastp.json"
```

Build the reference index:

```bash
salmon index \
    -t ref/transcripts.fa \
    -i ref/salmon_index \
    -p 16
```

Quantify:

```bash
mkdir -p output/salmon/"$SAMPLE"

salmon quant \
    -i ref/salmon_index \
    -l A \
    -r trimmed/"${SAMPLE}.trimmed.fastq.gz" \
    --validateMappings \
    --seqBias \
    --gcBias \
    -p 16 \
    --numBootstraps 100 \
    -o output/salmon/"$SAMPLE"
```

Final output:

```text
output/salmon/SRR6305029/quant.sf
```

---

# 19. Example complete paired-end workflow

Set the sample:

```bash
SAMPLE="SRRXXXXXXX"
```

Download:

```bash
prefetch "$SAMPLE" \
    --output-directory raw/sra
```

Convert:

```bash
fasterq-dump raw/sra/"$SAMPLE" \
    --split-files \
    --threads 16 \
    --temp output/tmp/fasterq \
    --outdir raw
```

Trim:

```bash
fastp \
    -i raw/"${SAMPLE}_1.fastq" \
    -I raw/"${SAMPLE}_2.fastq" \
    -o trimmed/"${SAMPLE}_R1.trimmed.fastq.gz" \
    -O trimmed/"${SAMPLE}_R2.trimmed.fastq.gz" \
    --qualified_quality_phred 20 \
    --length_required 30 \
    --thread 16 \
    --html trimmed/"${SAMPLE}_fastp.html" \
    --json trimmed/"${SAMPLE}_fastp.json"
```

Build the transcriptome index if it has not already been built:

```bash
salmon index \
    -t ref/transcripts.fa \
    -i ref/salmon_index \
    -p 16
```

Quantify:

```bash
mkdir -p output/salmon/"$SAMPLE"

salmon quant \
    -i ref/salmon_index \
    -l A \
    -1 trimmed/"${SAMPLE}_R1.trimmed.fastq.gz" \
    -2 trimmed/"${SAMPLE}_R2.trimmed.fastq.gz" \
    --validateMappings \
    --seqBias \
    --gcBias \
    -p 16 \
    --numBootstraps 100 \
    -o output/salmon/"$SAMPLE"
```

Final output:

```text
output/salmon/SRRXXXXXXX/quant.sf
```

---

# 20. Methods description

A concise Methods description matching this workflow is:

> RNA-seq reads were downloaded from the NCBI Sequence Read Archive using SRA Toolkit and converted to FASTQ format with `fasterq-dump`. Reads were quality filtered using fastp with a minimum Phred quality threshold of 20 and a minimum retained read length of 30 nucleotides. A Salmon index was constructed from the reference transcriptome, and transcript abundance was quantified using automatic library-type inference, mapping validation, sequence-bias correction, and GC-bias correction. Single-end libraries were supplied using the `-r` argument, whereas paired-end libraries were supplied using the `-1` and `-2` arguments. Transcript-level abundance estimates were obtained from the resulting `quant.sf` files.

For duplicate-gene analyses:

> Gene duplication categories were assigned independently of Salmon quantification and integrated with expression estimates during downstream analysis.

---

# 21. Reproducibility checklist

Record the following for every dataset:

- [ ] Species
- [ ] SRA accession
- [ ] Experimental condition
- [ ] Biological replicate
- [ ] Single-end or paired-end
- [ ] Read length
- [ ] Library strandedness
- [ ] Reference transcriptome
- [ ] Genome assembly
- [ ] Annotation release
- [ ] SRA Toolkit version
- [ ] fastp version
- [ ] Salmon version
- [ ] fastp parameters
- [ ] Salmon parameters
- [ ] Salmon mapping rate
- [ ] Inferred library type
- [ ] Final `quant.sf` location
- [ ] Downstream gene annotation
- [ ] Duplication-category annotation, if applicable

---

# 22. References

## Salmon

Patro R, Duggal G, Love MI, Irizarry RA, Kingsford C. 2017.  
**Salmon provides fast and bias-aware quantification of transcript expression.**  
*Nature Methods* 14:417–419.  
https://doi.org/10.1038/nmeth.4197

## Mapping and abundance estimation

Srivastava A, Malik L, Sarkar H, et al. 2020.  
**Alignment and mapping methodology influence transcript abundance estimation.**  
*Genome Biology* 21:239.  
https://doi.org/10.1186/s13059-020-02151-8

---

# Final workflow summary

```text
                 SRA accession
                      ↓
                   prefetch
                      ↓
                fasterq-dump
                      ↓
                   raw FASTQ
                      ↓
                    fastp
                      ↓
                trimmed FASTQ
                      ↓
             Salmon transcriptome
                    index
                      ↓
                salmon quant
                      ↓
                   quant.sf
```

For single-end libraries:

```bash
-r sample.fastq.gz
```

For paired-end libraries:

```bash
-1 sample_R1.fastq.gz
-2 sample_R2.fastq.gz
```

Salmon can incorporate ambiguous fragment-to-transcript assignments during transcript abundance estimation, which is useful when working with sequence-similar genes such as paralogs. Biological duplication categories are assigned separately during downstream analysis.

# vgp-genome-assembly-yeast
# 🧬 Genome Assembly using HiFi, Bionano, and Hi-C Data

**Course:** Bioinformatics / Genomics Lab  
**Tutorial source:** [Galaxy Training Network — VGP Genome Assembly](https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html)  
**Platform:** [Galaxy](https://usegalaxy.org)  
**Model organism:** *Saccharomyces cerevisiae* S288C (synthetic diploid reads)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Pipeline Summary](#pipeline-summary)
- [Tools Used](#tools-used)
- [Data](#data)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [1. Data Upload & Preprocessing](#1-data-upload--preprocessing)
  - [2. Genome Profile Analysis](#2-genome-profile-analysis)
  - [3. Assembly with hifiasm](#3-assembly-with-hifiasm)
  - [4. Assembly Evaluation](#4-assembly-evaluation)
  - [5. Scaffolding with Bionano](#5-scaffolding-with-bionano)
  - [6. Scaffolding with Hi-C (YaHS)](#6-scaffolding-with-hi-c-yahs)
  - [7. Final Evaluation](#7-final-evaluation)
- [Key Results](#key-results)
- [File Structure](#file-structure)
- [Glossary](#glossary)
- [References](#references)

---

## Overview

This repository documents the completion of the **Vertebrate Genomes Project (VGP) genome assembly tutorial** using the Galaxy bioinformatics platform. The goal of this tutorial is to assemble a high-quality, chromosome-level genome from raw sequencing reads using a multi-technology approach.

The pipeline combines three complementary sequencing technologies:

| Technology | Purpose |
|---|---|
| **PacBio HiFi** | Primary long-read assembly (10–25 kbp reads, ≥99% accuracy) |
| **Bionano optical maps** | Long-range scaffolding, resolves large structural regions |
| **Illumina Hi-C** | Chromosome-scale scaffolding using chromatin conformation |

The organism used is *Saccharomyces cerevisiae* S288C, a well-characterized yeast with a ~12 Mb haploid genome, making it ideal for benchmarking assembly quality.

---

## Pipeline Summary

```
Raw HiFi reads
      │
      ▼
[Cutadapt] — adapter removal
      │
      ▼
[Meryl + GenomeScope2] — k-mer profiling → genome size, heterozygosity
      │
      ▼
[hifiasm (Hi-C mode)] — contig assembly → Hap1 & Hap2 GFA graphs
      │
      ▼
[gfastats + BUSCO + Merqury] — assembly QC
      │
      ▼
[Bionano Hybrid Scaffold] — optical map scaffolding
      │
      ▼
[BWA-MEM2 + YaHS] — Hi-C scaffolding → chromosome-scale assembly
      │
      ▼
[PretextMap + Pretext Snapshot] — final Hi-C contact map QC
```

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Cutadapt | 4.4+galaxy0 | Adapter trimming / removal |
| Meryl | 1.3+galaxy6 | k-mer counting and histogram generation |
| GenomeScope2 | 2.0+galaxy2 | Genome size and heterozygosity estimation |
| hifiasm | 0.19.8+galaxy0 | De novo contig assembly |
| gfastats | 1.3.6+galaxy0 | Assembly statistics and GFA→FASTA conversion |
| BUSCO | 5.5.0+galaxy0 | Gene completeness assessment |
| Merqury | 1.3+galaxy3 | k-mer based assembly QV and completeness |
| Bionano Hybrid Scaffold | 3.7.0+galaxy3 | Optical map scaffolding |
| BWA-MEM2 | 2.2.1+galaxy1 | Hi-C read mapping |
| Filter and merge | 1.0+galaxy1 | Chimeric Hi-C read filtering |
| YaHS | 1.2a.2+galaxy1 | Hi-C scaffolding |
| PretextMap | 0.1.9+galaxy0 | Contact map generation |
| Pretext Snapshot | 0.0.3+galaxy1 | Contact map visualization |

---

## Data

All input data is publicly available from Zenodo.

### HiFi Reads (FASTA)
Synthetic 50× PacBio HiFi reads for *S. cerevisiae* S288C:

```
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_01.fasta
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_02.fasta
https://zenodo.org/record/6098306/files/HiFi_synthetic_50x_03.fasta
```

### Hi-C Reads (FASTQ)
Illumina paired-end Hi-C reads:

```
https://zenodo.org/record/5550653/files/SRR7126301_1.fastq.gz  (Forward)
https://zenodo.org/record/5550653/files/SRR7126301_2.fastq.gz  (Reverse)
```

### Bionano Optical Map (CMAP)
```
https://zenodo.org/records/5887339/files/bionano.cmap
```

---

## Step-by-Step Workflow

### 1. Data Upload & Preprocessing

**Objective:** Upload all datasets to Galaxy and remove adapter sequences from HiFi reads.

The three HiFi FASTA files were uploaded to Galaxy and grouped into a **dataset collection** named `HiFi data`. This collection format allows tools to process all files in parallel.

Cutadapt was run to **discard any HiFi reads containing adapter sequences** (not trim, but fully remove):

| Parameter | Value |
|---|---|
| Mode | Single-end |
| Adapter 1 | `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT` |
| Adapter 2 | `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT` |
| Max error rate | 0.1 |
| Min overlap | 35 |
| Reverse complement search | Yes |
| Discard trimmed reads | Yes |

> **Why discard instead of trim?** Because SMRT sequencing can embed adapters anywhere within a read (not just at the ends), any read containing an adapter is likely chimeric and should be removed entirely.

Output: `HiFi_collection (trimmed)`

---

### 2. Genome Profile Analysis

**Objective:** Estimate genome size, heterozygosity, and repeat content from k-mer frequencies before assembly.

#### 2a. Meryl — k-mer counting

Meryl was run three times:
1. **Count** k-mers (k=31) in each HiFi file separately (parallelized via collection)
2. **Merge** all counts with `Union-sum` into a single database
3. **Generate histogram** from the merged database

> k=31 was chosen as it is long enough to avoid most repetitive k-mers while remaining robust to sequencing errors.

#### 2b. GenomeScope2 — genome profiling

GenomeScope2 fit a diploid model (ploidy=2, k=31) to the k-mer histogram.

**Key results:**

| Parameter | Estimated Value |
|---|---|
| Haploid genome size | ~11.7 Mb |
| Heterozygosity | ~0.576% |
| Diploid coverage | ~50× |
| Haploid coverage | ~25× |
| Model fit | >93% |

The bimodal k-mer distribution (peaks at ~25× and ~50×) is consistent with a diploid genome. These values were used to parameterize downstream tools.

---

### 3. Assembly with hifiasm

**Objective:** Assemble phased haplotype contigs using HiFi + Hi-C reads.

**Mode used:** Hi-C phased mode (`hifiasm --hic`)

This mode uses Hi-C data from the same individual to phase contigs into two distinct haplotypes (Hap1 and Hap2) rather than producing a single pseudohaplotype assembly.

| Parameter | Value |
|---|---|
| Assembly mode | Standard |
| Input reads | HiFi_collection (trimmed) |
| Hi-C R1 | Hi-C_dataset_F |
| Hi-C R2 | Hi-C_dataset_R |

**Outputs:**
- `Hap1 contigs graph` (GFA format)
- `Hap2 contigs graph` (GFA format)

GFA files were converted to FASTA using gfastats (with path generation enabled):
- `Hap1 contigs FASTA`
- `Hap2 contigs FASTA`

---

### 4. Assembly Evaluation

**Objective:** Assess assembly quality using multiple complementary metrics.

#### 4a. gfastats — summary statistics

gfastats was run on both haplotype GFA files with expected genome size = **11,747,160 bp**.

| Statistic | Hap1 | Hap2 |
|---|---|---|
| # contigs | 16 | 17 |
| Total length | ~11.3 Mb | ~12.2 Mb |

> Note: Results may vary slightly between runs due to algorithmic stochasticity.

#### 4b. BUSCO — gene completeness

BUSCO was run against the **Saccharomycetes** lineage database using Metaeuk mode.

- Both haplotypes showed high completeness with very low duplication, indicating a clean assembly without major false duplications at this stage.

#### 4c. Merqury — k-mer based QV

Merqury compared k-mers in the HiFi reads against both haplotype assemblies.

- The **spectra-cn plot** showed diploid coverage of ~50×, consistent with GenomeScope2.
- The **spectra-asm plot** showed the haploid k-mers (~25×) split between Hap1 and Hap2, with homozygous k-mers (~50×) shared between both — the expected pattern for a well-phased diploid assembly.

---

### 5. Scaffolding with Bionano

**Objective:** Use optical map data to orient and order contigs into longer scaffolds, resolving regions difficult for sequence-based assembly alone.

The Bionano Hybrid Scaffold tool integrates in-silico sequence maps with Bionano genome maps via five steps:
1. Generate in silico sequence maps
2. Align against Bionano genome maps and resolve conflicts
3. Merge non-conflicting maps into hybrid scaffolds
4. Align sequence maps to hybrid scaffolds
5. Output AGP and FASTA files

**Parameters:**
- Configuration mode: VGP mode
- Genome maps conflict filter: Cut contig at conflict
- Sequences conflict filter: Cut contig at conflict

After scaffolding, the NGScontigs scaffold output and the unscaffolded contigs were concatenated into: `Hap1 assembly bionano`

gfastats was then run to evaluate the improvement in contiguity.

---

### 6. Scaffolding with Hi-C (YaHS)

**Objective:** Use chromatin conformation data to order and orient scaffolds to chromosome scale.

#### 6a. Pre-processing Hi-C reads

Hi-C reads were mapped **individually** (not as pairs) against `Hap1 assembly bionano` using BWA-MEM2, sorted by read name (QNAME), then merged and filtered for chimeric reads using the Filter and merge tool.

> Hi-C reads must be mapped individually because the insert size between read pairs can range from 1 bp to hundreds of Mb — violating the assumptions of standard paired-end aligners.

**Output:** `BAM Hi-C reads`

#### 6b. Generate initial contact map

PretextMap and Pretext Snapshot were used to generate a pre-scaffolding Hi-C contact map for comparison.

#### 6c. YaHS scaffolding

YaHS was run using the Bionano assembly as input contigs and the Hi-C BAM as the alignment file.

| Parameter | Value |
|---|---|
| Restriction enzyme | `CTTAAG` (HindIII) |

**Output:** `YaHS Scaffolds FASTA`

YaHS advantages over other Hi-C scaffolders:
- Does not require a pre-specified chromosome number
- Performs multi-round hierarchical scaffolding at decreasing resolutions
- Includes built-in assembly error correction based on Hi-C coverage

---

### 7. Final Evaluation

Hi-C reads were remapped to the YaHS scaffolds and a new contact map was generated for comparison.

**Key observations from final contact map:**
- 17 distinct scaffolds visible, corresponding to the 16 haploid chromosomes + mitochondrial DNA
- All contact signals concentrated along the diagonal, indicating correct contig ordering
- Differences between pre- and post-YaHS maps visible in circled regions, showing corrections of inversions made by Hi-C scaffolding

---

## Key Results

| Metric | Value |
|---|---|
| Estimated genome size | ~11.7 Mb |
| Final # scaffolds | ~17 (matches reference: 16 chr + mtDNA) |
| Assembly matches reference | Yes — contact maps nearly identical |
| BUSCO completeness | High (Saccharomycetes lineage) |
| Haplotype phasing | Hap1 / Hap2 via Hi-C mode |

The final assembly is essentially chromosome-level and closely matches the *S. cerevisiae* S288C reference genome in terms of sequence length, scaffold number, and Hi-C contact map topology.

---

## File Structure

```
vgp-genome-assembly/
├── README.md                    ← This file
├── Methods/
│   ├── 01_preprocessing.md      ← Cutadapt adapter removal details
│   ├── 02_kmer_profiling.md     ← Meryl + GenomeScope2 methods
│   ├── 03_hifiasm_assembly.md   ← hifiasm Hi-C mode details
│   ├── 04_evaluation.md         ← gfastats, BUSCO, Merqury
│   ├── 05_bionano.md            ← Bionano scaffolding
│   └── 06_hic_scaffolding.md    ← YaHS + Pretext
├── Results/
│   ├── genomescope_summary.txt  ← GenomeScope2 output (paste from Galaxy)
│   ├── gfastats_hap1_hap2.txt   ← Assembly stats table
│   ├── busco_summary_hap1.txt   ← BUSCO short summary
│   ├── busco_summary_hap2.txt
│   └── screenshots/             ← Contact map images, BUSCO plots, etc.
├── Parameters/
│   └── tool_parameters.md       ← All tool parameters in one place
├── Discussion/
│   └── Discussion.md       ← All tool parameters in one place
└── glossary.md                  ← Key terms reference
```

> **Note:** Raw sequencing data and large output files are not included in this repository. They are available from Zenodo (see [Data](#data) section). Galaxy history outputs should be downloaded and placed in the `results/` folder.

---

## Glossary

| Term | Definition |
|---|---|
| **Contig** | A contiguous, gap-free assembled sequence |
| **Scaffold** | One or more contigs joined by gap sequences (N's) |
| **Haplotype** | One version of a chromosome in a diploid organism |
| **Phasing** | Assigning contigs to their parental haplotype of origin |
| **HiFi reads** | PacBio reads 10–25 kbp long with ≥99% accuracy |
| **k-mer** | A substring of length k from a sequence |
| **N50** | Contig length where ≥50% of the assembly is in contigs of that size or longer |
| **BUSCO** | Benchmarking Universal Single-Copy Orthologs — gene completeness metric |
| **QV** | Quality value — phred-scale measure of assembly base accuracy |
| **Hi-C** | Chromatin conformation capture sequencing — captures 3D genome organization |
| **Optical map** | Long-range restriction enzyme-based physical map of the genome |
| **Purging** | Removing false duplications (haplotigs) from a primary assembly |
| **GFA** | Graphical Fragment Assembly format — encodes assembly graphs |
| **T2T** | Telomere-to-telomere — a completely gapless chromosome assembly |

---

## References

- Cheng, H. et al. (2021). Haplotype-resolved de novo assembly using phased assembly graphs with hifiasm. *Nature Methods*, 18, 170–175.
- Rhie, A. et al. (2020). Merqury: reference-free quality, completeness, and phasing assessment for genome assemblies. *Genome Biology*, 21, 245.
- Rhie, A. et al. (2021). Towards complete and error-free genome assemblies of all vertebrate species. *Nature*, 592, 737–746.
- Ranallo-Benavidez, T.R. et al. (2020). GenomeScope 2.0 and Smudgeplots for reference-free profiling of polyploid genomes. *Nature Communications*, 11, 1432.
- Simão, F.A. et al. (2015). BUSCO: assessing genome assembly and annotation completeness with single-copy orthologs. *Bioinformatics*, 31(19), 3210–3212.
- Wenger, A.M. et al. (2019). Accurate circular consensus long-read sequencing improves variant detection and assembly of a human genome. *Nature Biotechnology*, 37, 1155–1162.
- Zhou, C. et al. (2022). YaHS: yet another Hi-C scaffolding tool. *Bioinformatics*, 39(1).
- Formenti, G. et al. (2022). Gfastats: conversion, evaluation and manipulation of genome sequences using assembly graphs. *Bioinformatics*, 38(17), 4214–4216.
- Galaxy Training Network. VGP Genome Assembly Tutorial. https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html

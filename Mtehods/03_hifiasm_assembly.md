# 03 — Contig Assembly with hifiasm (Hi-C Mode)

## Objective

Assemble the filtered HiFi reads into fully phased haplotype contigs using hifiasm's Hi-C phased mode, producing two separate haplotype assemblies (Hap1 and Hap2) and converting the assembly graph output to FASTA format for downstream evaluation and scaffolding.

---

## Background: Hifiasm

Hifiasm is a fast, open-source de novo assembler developed specifically for PacBio HiFi reads. It builds an assembly graph from all-versus-all read overlaps, then resolves the graph into contigs. A key strength of hifiasm is its ability to distinguish near-identical sequences — such as the two haplotypes in a diploid organism, or diverged segmental duplications — by leveraging the high accuracy and length of HiFi reads together with phase information from Hi-C or parental data.

### Assembly Modes

Hifiasm supports three phasing modes:

| Mode | Input | Output | Notes |
|---|---|---|---|
| Solo | HiFi only | Primary + alternate (pseudohaplotype) | Requires purging; haplotypes mixed |
| Hi-C phased | HiFi + Hi-C | Hap1 + Hap2 | Phased using same-individual Hi-C; no purging usually needed |
| Trio | HiFi + parental reads | Maternal + paternal | Gold standard; requires parental samples |

**This tutorial used Hi-C phased mode**, as Hi-C data from the same individual was available. This mode uses Hi-C read pairs to identify which reads originate from the same chromosome, then partitions the assembly graph into two haplotype-specific subgraphs.

### How Hi-C Phasing Works in hifiasm

Hi-C reads capture pairs of genomic loci that are physically proximal in the nucleus. Within a chromosome, loci on the same haplotype are more likely to interact than loci on the homologous chromosome. Hifiasm uses this signal to assign "bubbles" in the assembly graph — nodes representing heterozygous sites — to one of the two haplotypes, thereby resolving the graph into two distinct haplotype paths.

---

## Tool: hifiasm

**Version:** 0.19.8+galaxy0  
**Galaxy tool ID:** `toolshed.g2.bx.psu.edu/repos/bgruening/hifiasm/hifiasm/0.19.8+galaxy0`

---

## Input

| File | Format | Description |
|---|---|---|
| `HiFi_collection (trimmed)` | FASTA collection | Adapter-filtered HiFi reads |
| `Hi-C_dataset_F` | FASTQ.gz | Forward Hi-C reads |
| `Hi-C_dataset_R` | FASTQ.gz | Reverse Hi-C reads |

---

## Parameters

| Parameter | Value | Rationale |
|---|---|---|
| Assembly mode | Standard | Standard diploid assembly (not inbred/homozygous mode) |
| Input reads | `HiFi_collection (trimmed)` | Adapter-cleaned HiFi reads |
| Hi-C partition | Specify | Enables Hi-C phased mode |
| Hi-C R1 | `Hi-C_dataset_F` | Forward Hi-C reads from the same individual |
| Hi-C R2 | `Hi-C_dataset_R` | Reverse Hi-C reads from the same individual |

---

## Output (GFA Format)

Hifiasm produces assembly graphs in **GFA (Graphical Fragment Assembly)** format rather than FASTA. GFA encodes the full assembly graph including sequences (S lines), edges/overlaps (L lines), and paths (P lines). This is more information-rich than FASTA and preserves the graph topology.

| Output | Renamed to | Tag |
|---|---|---|
| Hi-C hap1 balanced contig graph | `Hap1 contigs graph` | `#hap1` |
| Hi-C hap2 balanced contig graph | `Hap2 contigs graph` | `#hap2` |

---

## GFA to FASTA Conversion

Downstream QC tools (BUSCO, Merqury) require FASTA format, so the GFA outputs were converted using gfastats.

### Tool: gfastats

**Version:** 1.3.6+galaxy0

| Parameter | Value |
|---|---|
| Input | `Hap1 contigs graph` + `Hap2 contigs graph` |
| Tool mode | Genome assembly manipulation |
| Output format | FASTA |
| Generate initial set of paths | Yes (toggled on) |

The "generate initial set of paths" option instructs gfastats to traverse the GFA graph and output each path (contig) as a linear FASTA sequence.

**Outputs:**
- `Hap1 contigs FASTA`
- `Hap2 contigs FASTA`

---

## Assembly Results (Pre-scaffolding)

| Statistic | Hap1 | Hap2 |
|---|---|---|
| Number of contigs | 16 | 17 |
| Total length | ~11.3 Mb | ~12.2 Mb |
| Expected genome size | 11.7 Mb | 11.7 Mb |

Both haplotypes assembled into a small number of contigs closely matching the expected genome size, which is a strong indicator of successful assembly given that *S. cerevisiae* has 16 chromosomes. The slight difference in total length between haplotypes is normal and reflects asymmetric resolution of heterozygous regions.

---

## Why Hi-C Mode vs. Solo Mode?

| Consideration | Solo mode | Hi-C mode |
|---|---|---|
| Phasing quality | Pseudohaplotype (switch errors possible) | True haplotype phasing |
| Purging required | Usually yes | Usually not |
| Data requirement | HiFi only | HiFi + Hi-C from same individual |
| Output | Primary + alternate | Hap1 + Hap2 |
| Best for | When Hi-C unavailable | When Hi-C from same individual is available |

Since Hi-C data from the same *S. cerevisiae* individual was available, Hi-C mode was the appropriate choice. It produces two balanced haplotype assemblies without the switch errors characteristic of pseudohaplotype assembly, and eliminates the need for a separate purging step.

---

## Notes

- The Hi-C reads used for phasing **must come from the same individual** as the HiFi reads. Using Hi-C data from a different individual of the same species would provide no phasing information.
- Hi-C data from the same individual can still be used for **scaffolding** even if it cannot be used for phasing (i.e., if sequenced from a different individual), but phasing in hifiasm would not be possible in that case.
- Hifiasm does not require the number of expected chromosomes as input — it infers the graph structure directly from the data.

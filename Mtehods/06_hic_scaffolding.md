# 06 — Hi-C Scaffolding with YaHS and Contact Map Visualisation

## Objective

Use Hi-C chromatin conformation data to order and orient the Bionano-scaffolded contigs into chromosome-scale scaffolds, then evaluate the result by comparing Hi-C contact maps before and after scaffolding.

---

## Background: Hi-C Sequencing

Hi-C is a sequencing-based molecular assay that captures the three-dimensional organisation of chromatin in the nucleus. It works by crosslinking DNA that is physically proximate in space, proximity-ligating the crosslinked ends, and then sequencing the resulting chimeric fragments. Because genomic loci on the same chromosome are physically closer in the nucleus than loci on different chromosomes, Hi-C reads provide a strong signal for:

1. **Which contigs belong to the same chromosome** (high inter-contig contact frequency)
2. **The order and orientation of contigs along a chromosome** (contacts decay with linear distance)
3. **Chromosome boundaries** (sharp drops in inter-contig contact frequency)

### Contact Map Interpretation

A Hi-C contact map is a symmetric matrix where each cell (i, j) represents the frequency of contacts observed between loci i and j. Two hallmark patterns are expected in a correctly assembled genome:

- **Diagonal blocks:** Regions of high contact along the diagonal correspond to individual chromosomes — loci on the same chromosome interact far more than loci on different chromosomes
- **Distance decay:** Contact frequency decreases as linear distance along the chromosome increases

Misassemblies appear as off-diagonal signals — high contact frequency between regions that are placed far apart in the assembly, indicating they should actually be adjacent.

---

## Step 1: Hi-C Read Mapping with BWA-MEM2

### Why Map Reads Individually?

Standard paired-end aligners assume the insert size between read pairs follows a known distribution (typically a few hundred base pairs for Illumina data). Hi-C insert sizes violate this assumption completely — the "insert" is actually the distance between two loci in 3D space, which can range from 1 bp to hundreds of megabases. Mapping Hi-C reads as pairs would cause most aligners to discard or mismapped the majority of reads.

Solution: map each read independently, then merge the resulting alignments.

### Tool: BWA-MEM2

**Version:** 2.2.1+galaxy1  
BWA-MEM2 is an optimised implementation of the BWA-MEM algorithm with improved performance for large genomes. It was run twice — once for the forward reads and once for the reverse reads.

#### Forward reads mapping

| Parameter | Value |
|---|---|
| Reference | `Hap1 assembly bionano` (index built from history) |
| Reads | Single-end |
| Input | `Hi-C_dataset_F` |
| Read groups | Do not set |
| Analysis mode | 1. Simple Illumina mode |
| BAM sorting | Sort by read names (QNAME) |

**Output:** `BAM forward`

#### Reverse reads mapping

Identical parameters, with `Hi-C_dataset_R` as input.

**Output:** `BAM reverse`

> **QNAME sorting** is required by the Filter and merge step, which needs to match forward and reverse reads by their read name.

---

## Step 2: Filter and Merge Chimeric Reads

### Tool: Filter and merge (Arima Genomics)

**Version:** 1.0+galaxy1

This tool implements the Arima Genomics Hi-C read filtering protocol. It:
1. Merges the separately mapped forward and reverse BAM files
2. Filters out chimeric read pairs (where the two ends map to inconsistent locations, likely due to ligation artefacts)
3. Retains only valid Hi-C read pairs for downstream use

| Parameter | Value |
|---|---|
| First set of reads | `BAM forward` |
| Second set of reads | `BAM reverse` |

**Output:** `BAM Hi-C reads`

---

## Step 3: Generate Initial Contact Map (Pre-scaffolding)

A contact map was generated before YaHS scaffolding to provide a baseline for comparison.

### Tool: PretextMap

**Version:** 0.1.9+galaxy0  
Converts a BAM file of Hi-C alignments into a Pretext format contact map.

| Parameter | Value |
|---|---|
| Input | `BAM Hi-C reads` |
| Sort by | Don't sort |

**Output:** `PretextMap output`

### Tool: Pretext Snapshot

**Version:** 0.0.3+galaxy1  
Renders the Pretext contact map as a static image.

| Parameter | Value |
|---|---|
| Input | `PretextMap output` |
| Output format | PNG |
| Show grid | Yes |

The resulting image shows the contact map of the Bionano-scaffolded assembly, with 17 scaffolds visible as distinct triangular blocks.

---

## Step 4: YaHS Scaffolding

### Tool: YaHS

**Version:** 1.2a.2+galaxy1  
YaHS (Yet Another Hi-C Scaffolding tool) uses Hi-C contact information to order and orient assembled sequences into chromosome-scale scaffolds. Unlike most Hi-C scaffolders, YaHS does not require the user to specify the expected number of chromosomes.

### YaHS Algorithm Overview

YaHS operates in multiple rounds of hierarchical scaffolding at decreasing resolutions:

1. **Contact matrix construction:** Each contig is split into fixed-size bins. A contact matrix is built counting Hi-C read pairs linking each bin pair.
2. **Normalisation:** Contact frequencies are normalised by the number of restriction enzyme cut sites in each bin, correcting for sequence composition bias.
3. **Scaffolding graph:** A graph is constructed with contigs as nodes and weighted edges representing joining scores based on normalised inter-contig contact frequencies.
4. **Graph simplification:** Repeated rounds of edge filtering, tip trimming, repeat resolution, and bubble resolution simplify the graph.
5. **Path traversal:** The simplified graph is traversed to build scaffold sequences.
6. **Error correction:** Scaffolds are broken at positions with insufficient Hi-C coverage, which may represent assembly errors or misjoins.

This process repeats with increasing bin sizes until no further joins can be made.

### Parameters

| Parameter | Value | Rationale |
|---|---|---|
| Input contig sequences | `Hap1 assembly bionano` | Bionano-scaffolded contigs as the starting point |
| Hi-C alignment file | `BAM Hi-C reads` | Valid Hi-C read pairs mapped to the assembly |
| Restriction enzyme | Enter specific sequence | Use the actual enzyme, not a generic option |
| Restriction enzyme sequence | `CTTAAG` | HindIII recognition sequence — the enzyme used in this Hi-C library preparation |

> The restriction enzyme sequence `CTTAAG` corresponds to **HindIII**, a commonly used restriction enzyme in Hi-C library preparation. This information is required for YaHS to correctly normalise contact frequencies by cut site density.

**Output:** `YaHS Scaffolds FASTA`

---

## Step 5: Final Contact Map (Post-scaffolding)

The Hi-C read mapping, filtering, PretextMap, and Pretext Snapshot steps were repeated on the YaHS scaffolded assembly to produce a final contact map for comparison.

| Step | Input | Output |
|---|---|---|
| BWA-MEM2 (forward) | `Hi-C_dataset_F` vs `YaHS Scaffolds FASTA` | `BAM forward YaHS` |
| BWA-MEM2 (reverse) | `Hi-C_dataset_R` vs `YaHS Scaffolds FASTA` | `BAM reverse YaHS` |
| Filter and merge | `BAM forward YaHS` + `BAM reverse YaHS` | `BAM Hi-C reads YaHS` |
| PretextMap | `BAM Hi-C reads YaHS` | `PretextMap output YaHS` |
| Pretext Snapshot | `PretextMap output YaHS` | Final contact map PNG |

---

## Results and Contact Map Comparison

### Pre-YaHS (Bionano-scaffolded assembly)
- 17 scaffolds visible as distinct blocks
- Most contact signal along the diagonal
- Some off-diagonal signal indicating ordering/orientation issues in a few regions

### Post-YaHS (Final assembly)
- 17 scaffolds maintained
- Cleaner diagonal — off-diagonal signal resolved
- Inversion corrections visible in regions circled during tutorial review

The inversions corrected by YaHS were visible as regions where the pre-YaHS contact map showed off-diagonal signal (contacts between regions placed far apart in the assembly) that disappeared in the post-YaHS map after YaHS reoriented the relevant contigs.

---

## Final Assembly Statistics

| Metric | Value |
|---|---|
| Number of scaffolds | ~17 |
| Matches reference chromosome count | Yes (16 chr + mtDNA) |
| Contact map topology | Clean diagonal blocks |
| Coverage of expected genome size | ~100% |

The final assembly closely matches the *S. cerevisiae* S288C reference genome in both sequence content and three-dimensional chromatin organisation as captured by the Hi-C contact map.

---

## Notes

- **Manual curation** is the next step in a production VGP pipeline. A human curator would review the final contact map in the full Pretext viewer (not just a snapshot), identify any remaining off-diagonal signals or misassemblies, and make manual corrections. This step was not performed in this tutorial but would be essential for a publication-quality assembly.
- **Mitochondrial DNA** is typically assembled separately (VGP Workflow 0) and decontamination (Workflow 9) would be run after scaffolding to remove non-nuclear sequences from the final assembly. These steps were not covered in this tutorial.
- The restriction enzyme sequence must match what was used in the actual Hi-C wet-lab experiment. Using an incorrect enzyme would cause YaHS to normalise contact frequencies incorrectly, potentially degrading scaffolding quality.

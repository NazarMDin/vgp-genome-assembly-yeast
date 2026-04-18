# 04 — Assembly Evaluation

## Objective

Assess the quality of the Hap1 and Hap2 contig assemblies using three complementary, reference-free evaluation approaches: summary statistics (gfastats), gene completeness (BUSCO), and k-mer based accuracy and completeness (Merqury). Each tool captures a different dimension of assembly quality and together they provide a comprehensive picture of the assembly before scaffolding begins.

---

## Why Multiple Metrics?

No single metric captures all aspects of assembly quality:

| Metric | What it measures | What it misses |
|---|---|---|
| N50 / contig count | Contiguity | Sequence accuracy, completeness |
| BUSCO | Gene space completeness | Non-genic regions, accuracy |
| Merqury QV | Base-level accuracy, k-mer completeness | Gene content, contiguity |

Using all three together gives a much more reliable assessment than any single metric alone.

---

## Tool 1: gfastats — Summary Statistics

### Tool

**gfastats** v1.3.6+galaxy0  
A toolkit for manipulation and evaluation of FASTA and GFA assembly files. In this step it is used to generate standard assembly summary statistics.

### Parameters

| Parameter | Value |
|---|---|
| Input | `Hap1 contigs graph` + `Hap2 contigs graph` (GFA files) |
| Tool mode | Summary statistics generation |
| Expected genome size | 11747160 |
| Thousands separator | No |

The GFA files are used (rather than FASTA) so that graph-specific statistics can also be reported.

### Post-processing

The individual stats files for Hap1 and Hap2 were:
1. Joined side-by-side using **Column join** to allow direct comparison
2. Filtered to remove scaffold-related rows (no scaffolds exist at this stage) using **Search in textfiles** (exclude lines matching "scaffold")

**Final output:** `gfastats on hap1 and hap2 contigs`

### Statistics Explained

| Statistic | Definition |
|---|---|
| # contigs | Total number of contiguous sequences in the assembly |
| Total contig length | Sum of all contig lengths in base pairs |
| Largest contig | Length of the single longest contig |
| N50 | Contig length at which ≥50% of the total assembly is in contigs of that size or larger |
| NG50 | Like N50 but calculated relative to the expected genome size (11.7 Mb) rather than the assembly size |
| auN | Area under the Nx curve — a single number summarising the full Nx profile |
| GC content | Percentage of bases that are G or C |

### Results

| Statistic | Hap1 | Hap2 |
|---|---|---|
| # contigs | 16 | 17 |
| Total length | ~11.3 Mb | ~12.2 Mb |

The low contig count (16–17) relative to the 16 chromosomes of *S. cerevisiae* indicates that hifiasm successfully assembled most chromosomes as single, unbroken contigs — an excellent result for a eukaryotic genome at this stage.

---

## Tool 2: BUSCO — Gene Completeness Assessment

### Tool

**BUSCO** v5.5.0+galaxy0  
Benchmarking Universal Single-Copy Orthologs. BUSCO assesses assembly completeness by checking for the presence of genes that are expected to be present in exactly one copy per haploid genome in a given taxonomic clade. These genes are conserved across many species and their presence, absence, or duplication can reveal whether an assembly is complete, fragmented, or contains false duplications.

### Parameters

| Parameter | Value | Rationale |
|---|---|---|
| Input sequences | `Hap1 contigs FASTA` + `Hap2 contigs FASTA` | Both haplotypes evaluated separately |
| Lineage data source | Use cached lineage data | Avoids downloading the database each run |
| Cached database | Busco v5 Lineage Datasets | Full v5 BUSCO lineage collection |
| Mode | Genome assemblies (DNA) | Evaluating genomic DNA assembly (not proteins or transcripts) |
| Gene predictor | Use Metaeuk | Metaeuk is the recommended predictor for genome mode in BUSCO v5 |
| Lineage | Saccharomycetes | The appropriate clade for *S. cerevisiae* |
| Outputs generated | Short summary text + summary image | Provides both numeric and visual output |

> **Note for vertebrate genomes:** Replace "Saccharomycetes" with the appropriate vertebrate clade (e.g. Vertebrata, Tetrapoda, Actinopterygii) when running on non-yeast organisms.

### BUSCO Categories

Each gene in the BUSCO database is classified into one of five categories per assembly:

| Category | Abbreviation | Meaning |
|---|---|---|
| Complete, single-copy | C(S) | Present once — ideal |
| Complete, duplicated | C(D) | Present more than once — may indicate false duplication |
| Fragmented | F | Partially present — may indicate assembly gaps or errors |
| Missing | M | Not found — may indicate missing sequence |
| Total | T | Total BUSCO genes in the lineage dataset |

### Results

Both Hap1 and Hap2 showed high completeness with very low duplication, indicating:
- The assembly captures the expected gene content
- Haplotype phasing successfully separated the two copies of each gene into different assemblies (low duplication)
- No major assembly gaps affecting gene-coding regions (low fragmented/missing)

The low duplication level is particularly meaningful here — if the Hi-C phasing had failed, both haplotypes of each gene would appear in both assemblies, causing elevated C(D) scores.

---

## Tool 3: Merqury — K-mer Based Evaluation

### Tool

**Merqury** v1.3+galaxy3  
Merqury evaluates assembly quality by comparing k-mers in the raw read set against k-mers in the assembly. This approach is reference-free and measures two key properties: base-level accuracy (QV) and sequence completeness (what fraction of the read k-mers are represented in the assembly).

### Parameters

| Parameter | Value |
|---|---|
| Evaluation mode | Default mode |
| k-mer counts database | `Merged meryldb` (from Meryl step) |
| Number of assemblies | Two assemblies |
| First assembly | `Hap1 contigs FASTA` |
| Second assembly | `Hap2 contigs FASTA` |

### Outputs

Merqury generates three output collections:

| Collection | Contents |
|---|---|
| Stats | Completeness statistics per assembly |
| Plots | CN spectrum plots (spectra-cn, spectra-asm) |
| QV stats | Quality value estimates per assembly and per contig |

### Interpreting the CN Spectrum Plots

**spectra-cn plot (copy number spectrum):**

Tracks the multiplicity of each k-mer in the HiFi reads and colours it by how many times it appears in the combined assembly. Ideally:
- The grey region (k-mers only in reads, absent from assembly) should be small — indicates low missing sequence
- Red region (1-copy k-mers in assembly) should match the haploid coverage peak (~25×)
- Blue region (2-copy k-mers in assembly) should match the diploid coverage peak (~50×)

**spectra-asm plot (assembly-specific spectrum):**

Tracks which assembly (Hap1, Hap2, or both) contains each k-mer. Ideally:
- Homozygous k-mers (~50× coverage) should be in the "shared" (green) category — present in both haplotypes
- Haploid k-mers (~25× coverage) should be split between Hap1-only and Hap2-only — indicating clean haplotype separation

### Results Interpretation

The spectra-cn plot confirmed diploid coverage of ~50×, consistent with GenomeScope2. The grey region was minimal, indicating very high assembly completeness.

The spectra-asm plot showed:
- The ~50× homozygous k-mers predominantly in the shared (green) category — correctly assigned to both haplotypes
- The ~25× haplotype-specific k-mers split between Hap1 and Hap2 — confirming that phasing distributed alleles to the appropriate haplotypes

This pattern is the expected result of successful Hi-C phased assembly. An unphased or poorly phased assembly would show the haploid k-mers largely in the shared category (both haplotypes mixed into one assembly), which would also manifest as elevated BUSCO duplication.

### QV (Quality Value)

Merqury calculates a QV score for each assembly based on the fraction of assembly k-mers that are supported by the read k-mer database. QV is expressed on a Phred scale:

| QV | Per-base error rate | Accuracy |
|---|---|---|
| 30 | 1 in 1,000 | 99.9% |
| 40 | 1 in 10,000 | 99.99% |
| 50 | 1 in 100,000 | 99.999% |

HiFi-based assemblies typically achieve QV40–QV50, reflecting the high accuracy of HiFi reads.

---

## Summary

| Evaluation | Tool | Key result |
|---|---|---|
| Contiguity | gfastats | 16–17 contigs, total ~11–12 Mb |
| Gene completeness | BUSCO | High completeness, low duplication |
| Base accuracy | Merqury QV | High (QV40+ expected) |
| K-mer completeness | Merqury spectra | Minimal missing k-mers |
| Phasing quality | Merqury spectra-asm | Haploid k-mers correctly split between haplotypes |

All three tools indicated a high-quality, well-phased assembly ready to proceed to scaffolding.

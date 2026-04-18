# Tool Parameters Reference

All tool parameters used in the VGP genome assembly tutorial, in order of execution.

---

## 1. Cutadapt — Adapter Removal

| Parameter | Value |
|---|---|
| Galaxy version | 4.4+galaxy0 |
| Mode | Single-end |
| Input | HiFi_collection (FASTA collection) |
| Adapter 1 name | First adapter |
| Adapter 1 sequence | `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT` |
| Adapter 2 name | Second adapter |
| Adapter 2 sequence | `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT` |
| Maximum error rate | 0.1 |
| Minimum overlap length | 35 |
| Look in reverse complement | Yes |
| Discard trimmed reads | Yes |

**Output:** `HiFi_collection (trimmed)`

---

## 2. Meryl — k-mer Counting (Run 1: Count)

| Parameter | Value |
|---|---|
| Galaxy version | 1.3+galaxy6 |
| Operation | Count: count occurrences of canonical k-mers |
| Input | HiFi_collection (trimmed) |
| k-mer size | 31 |

**Output:** `meryldb` (collection)

---

## 3. Meryl — k-mer Merge (Run 2: Union-sum)

| Parameter | Value |
|---|---|
| Galaxy version | 1.3+galaxy6 |
| Operation | Union-sum |
| Input | Collection meryldb |

**Output:** `Merged meryldb`

---

## 4. Meryl — Histogram (Run 3)

| Parameter | Value |
|---|---|
| Galaxy version | 1.3+galaxy6 |
| Operation | Generate histogram dataset |
| Input | Merged meryldb |

**Output:** `meryldb histogram`

---

## 5. GenomeScope2

| Parameter | Value |
|---|---|
| Galaxy version | 2.0+galaxy2 |
| Input histogram | meryldb histogram |
| Ploidy | 2 |
| k-mer length | 31 |
| Output summary | Yes |
| Create testing.tsv | Yes |

**Output:** Linear plot, log plot, transformed plots, model file, summary

---

## 6. hifiasm — Hi-C Phased Assembly

| Parameter | Value |
|---|---|
| Galaxy version | 0.19.8+galaxy0 |
| Assembly mode | Standard |
| Input reads | HiFi_collection (trimmed) |
| Hi-C partition | Specify |
| Hi-C R1 | Hi-C_dataset_F |
| Hi-C R2 | Hi-C_dataset_R |

**Outputs:**
- `Hap1 contigs graph` (GFA) — tagged `#hap1`
- `Hap2 contigs graph` (GFA) — tagged `#hap2`

---

## 7. gfastats — GFA to FASTA Conversion

| Parameter | Value |
|---|---|
| Galaxy version | 1.3.6+galaxy0 |
| Input | Hap1 contigs graph + Hap2 contigs graph |
| Tool mode | Genome assembly manipulation |
| Output format | FASTA |
| Generate initial set of paths | Yes (toggled on) |

**Outputs:** `Hap1 contigs FASTA`, `Hap2 contigs FASTA`

---

## 8. gfastats — Assembly Statistics

| Parameter | Value |
|---|---|
| Galaxy version | 1.3.6+galaxy0 |
| Input | Hap1 contigs graph + Hap2 contigs graph |
| Tool mode | Summary statistics generation |
| Expected genome size | 11747160 |
| Thousands separator | No |

**Outputs:** `Hap1 stats`, `Hap2 stats`

Post-processing: Column join → filter (exclude lines containing "scaffold") → `gfastats on hap1 and hap2 contigs`

---

## 9. BUSCO

| Parameter | Value |
|---|---|
| Galaxy version | 5.5.0+galaxy0 |
| Input sequences | Hap1 contigs FASTA + Hap2 contigs FASTA |
| Lineage data source | Use cached lineage data |
| Cached database | Busco v5 Lineage Datasets |
| Mode | Genome assemblies (DNA) |
| Gene predictor | Use Metaeuk |
| Auto-detect lineage | No — select manually |
| Lineage | Saccharomycetes |
| Outputs | Short summary text + summary image |

**Outputs:** `BUSCO hap1`, `BUSCO hap2`

---

## 10. Merqury

| Parameter | Value |
|---|---|
| Galaxy version | 1.3+galaxy3 |
| Evaluation mode | Default mode |
| k-mer counts database | Merged meryldb |
| Number of assemblies | Two assemblies |
| First assembly | Hap1 contigs FASTA |
| Second assembly | Hap2 contigs FASTA |

**Outputs:** stats collection, plots collection (spectra-cn, spectra-asm), QV stats collection

---

## 11. Bionano Hybrid Scaffold

| Parameter | Value |
|---|---|
| Galaxy version | 3.7.0+galaxy3 |
| NGS FASTA | Hap1 contigs FASTA |
| BioNano CMAP | bionano.cmap |
| Configuration mode | VGP mode |
| Genome maps conflict filter | Cut contig at conflict |
| Sequences conflict filter | Cut contig at conflict |

Post-processing: Concatenate `NGScontigs scaffold NCBI trimmed` + `NGScontigs not scaffolded trimmed` → `Hap1 assembly bionano`

---

## 12. gfastats — Bionano Assembly Statistics

| Parameter | Value |
|---|---|
| Galaxy version | 1.3.6+galaxy0 |
| Input | Hap1 assembly bionano |
| Expected genome size | 11747160 |

**Output:** `Bionano stats`

---

## 13. BWA-MEM2 — Forward Hi-C Mapping

| Parameter | Value |
|---|---|
| Galaxy version | 2.2.1+galaxy1 |
| Reference | Hap1 assembly bionano (from history) |
| Reads | Single-end |
| Input | Hi-C_dataset_F |
| Read groups | Do not set |
| Analysis mode | 1. Simple Illumina mode |
| BAM sorting | Sort by read names (QNAME) |

**Output:** `BAM forward`

---

## 14. BWA-MEM2 — Reverse Hi-C Mapping

Same parameters as above, but:
- Input: `Hi-C_dataset_R`
- Output: `BAM reverse`

---

## 15. Filter and Merge — Chimeric Read Filtering

| Parameter | Value |
|---|---|
| Galaxy version | 1.0+galaxy1 |
| First set of reads | BAM forward |
| Second set of reads | BAM reverse |

**Output:** `BAM Hi-C reads`

---

## 16. PretextMap — Initial Contact Map

| Parameter | Value |
|---|---|
| Galaxy version | 0.1.9+galaxy0 |
| Input | BAM Hi-C reads |
| Sort by | Don't sort |

**Output:** `PretextMap output`

---

## 17. Pretext Snapshot — Initial Contact Map Image

| Parameter | Value |
|---|---|
| Galaxy version | 0.0.3+galaxy1 |
| Input | PretextMap output |
| Output format | PNG |
| Show grid | Yes |

---

## 18. YaHS — Hi-C Scaffolding

| Parameter | Value |
|---|---|
| Galaxy version | 1.2a.2+galaxy1 |
| Input contig sequences | Hap1 assembly bionano |
| Hi-C alignment file | BAM Hi-C reads |
| Restriction enzyme | Enter specific sequence |
| Restriction enzyme sequence | `CTTAAG` |

**Output:** `YaHS Scaffolds FASTA`

---

## 19–22. Final Contact Map (post-YaHS)

Steps 13–17 were repeated using `YaHS Scaffolds FASTA` as the reference:

- BWA-MEM2 forward → `BAM forward YaHS`
- BWA-MEM2 reverse → `BAM reverse YaHS`
- Filter and merge → `BAM Hi-C reads YaHS`
- PretextMap → `PretextMap output YaHS`
- Pretext Snapshot → Final contact map PNG

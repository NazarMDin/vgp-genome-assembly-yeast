# 02 — Genome Profile Analysis with Meryl and GenomeScope2

## Objective

Before assembling a genome, it is essential to estimate its key properties — size, heterozygosity, repeat content, and sequencing error rate — directly from the raw reads. This provides a prior expectation against which the final assembly can be evaluated, and produces parameters needed to configure downstream tools (particularly the expected genome size used by gfastats and purge_dups).

This is done by analysing the **k-mer frequency spectrum** of the reads. The distribution of how often each unique k-mer appears in the read set reflects the underlying genome structure in a predictable way.

---

## Background: K-mer Frequency Analysis

A k-mer is any substring of exactly k nucleotides. For a genome of size G sequenced at coverage C, the expected frequency of any unique k-mer in the reads is approximately C. Heterozygous positions generate two versions of the k-mers that span them — one from each haplotype — each appearing at approximately C/2 frequency. This produces the characteristic bimodal distribution seen in diploid genomes:

- **First peak (~C/2):** k-mers unique to one haplotype (heterozygous regions)
- **Second peak (~C):** k-mers shared by both haplotypes (homozygous regions)
- **Low-frequency k-mers (frequency 1–3):** Sequencing errors — random errors generate unique k-mers not present in the genome

The total area under the histogram is proportional to genome size, and the ratio of the two peak heights reflects the heterozygosity rate.

---

## Step 1: Meryl — K-mer Counting

### Tool

**Meryl** v1.3+galaxy6  
Meryl is a k-mer counting and database manipulation toolkit developed as part of the Celera/Verkko assembler ecosystem. It stores k-mers in lexicographic order in a binary database format optimised for downstream merging and querying.

### Run 1 — Count k-mers per file

Meryl was first run on each file in the HiFi collection separately (parallelised via Galaxy's collection mechanism), counting the occurrence of every canonical k-mer of length 31.

| Parameter | Value | Rationale |
|---|---|---|
| Operation | Count canonical k-mers | Canonical = the lexicographically smaller of a k-mer and its reverse complement, avoids double-counting |
| Input | `HiFi_collection (trimmed)` | Each file processed independently in parallel |
| k-mer size | 31 | See note below |

**Output:** `meryldb` (collection of per-file k-mer databases)

> **Why k=31?** A k-mer of length 31 is long enough that most k-mers are unique in a typical eukaryotic genome (the probability of a random match decreases exponentially with k), while being short enough to be robust to sequencing errors (a single error only corrupts the k-mers overlapping it, not distant ones). For very large or highly repetitive genomes (>10 Gb), larger k values (e.g. 51) are recommended. For *S. cerevisiae* at ~12 Mb, k=31 is appropriate.

### Run 2 — Merge databases

The per-file k-mer databases were merged into a single database using the Union-sum operation, which combines k-mer counts across all input files by summing their frequencies.

| Parameter | Value |
|---|---|
| Operation | Union-sum |
| Input | `meryldb` (collection) |

**Output:** `Merged meryldb`

### Run 3 — Generate histogram

The merged database was converted to a frequency histogram suitable for GenomeScope2.

| Parameter | Value |
|---|---|
| Operation | Generate histogram dataset |
| Input | `Merged meryldb` |

**Output:** `meryldb histogram`

The histogram has two columns: k-mer frequency (x-axis) and the number of distinct k-mers observed at that frequency (y-axis).

---

## Step 2: GenomeScope2 — Genome Modelling

### Tool

**GenomeScope2** v2.0+galaxy2  
GenomeScope2 fits a mixture of negative binomial distributions to the k-mer frequency histogram using nonlinear least-squares optimisation. It extends the original GenomeScope model to support polyploid genomes and produces statistically rigorous estimates of genome properties.

### Parameters

| Parameter | Value | Rationale |
|---|---|---|
| Input histogram | `meryldb histogram` | Output from Meryl run 3 |
| Ploidy | 2 | *S. cerevisiae* is diploid |
| k-mer length | 31 | Must match the k used in Meryl |
| Output summary | Yes | Produces the text summary of estimated parameters |
| Create testing.tsv | Yes | Outputs raw model parameters for reference |

### Outputs

GenomeScope2 produces six outputs:

| Output | Description |
|---|---|
| Linear plot | k-mer spectrum with fitted model, frequency vs coverage |
| Log plot | Logarithmic version of the linear plot |
| Transformed linear plot | frequency × coverage vs coverage — enhances higher-order peaks |
| Transformed log plot | Log version of the transformed plot |
| Model file | Detailed model fitting report |
| Summary | Estimated genome parameters |

---

## Results

| Parameter | Estimated Value |
|---|---|
| Haploid genome size | ~11.7 Mb |
| Heterozygosity | ~0.576% |
| Diploid sequencing coverage | ~50× |
| Haploid sequencing coverage | ~25× |
| Sequencing error rate | Low |
| Model fit | >93% |

### Interpretation

The k-mer spectrum showed a clear bimodal distribution with peaks at ~25× and ~50× coverage — the expected pattern for a diploid genome sequenced at 50× diploid depth. The first peak represents k-mers from heterozygous regions (present in only one haplotype, so seen at half the diploid coverage), and the second represents k-mers from homozygous regions (present in both haplotypes, seen at full diploid coverage).

The model fit of >93% indicates that the observed data closely follows the theoretical diploid model, validating the estimated parameters. The low heterozygosity (~0.576%) is consistent with the known biology of the S288C laboratory strain, which has been maintained under controlled conditions and has accumulated relatively few polymorphisms.

The estimated haploid genome size of ~11.7 Mb is close to the known *S. cerevisiae* reference genome size of ~12.1 Mb — a difference of ~3%, which is within acceptable range given that k-mer based estimates exclude low-coverage regions and are affected by repeat collapsing.

---

## Key Value for Downstream Tools

The estimated haploid genome size — **11,747,160 bp** — was used as the `expected genome size` parameter in all subsequent gfastats runs to calculate NG50 and other reference-based statistics.

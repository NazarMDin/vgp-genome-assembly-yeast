# 05 — Hybrid Scaffolding with Bionano Optical Maps

## Objective

Use long-range physical map information from Bionano optical maps to order and orient the assembled contigs into longer scaffolds, resolving structural regions and estimating gap sizes that sequence-based assembly alone cannot address.

---

## Background: Bionano Optical Mapping

Bionano optical mapping is a technology that produces genome-scale physical maps by labelling long, intact DNA molecules at specific sequence motifs and imaging them on a nanochannel array. The resulting data encodes the positions of restriction sites (or nick sites, depending on the enzyme) along single DNA molecules that can be hundreds of kilobases to megabases in length.

### Key Advantages

- **Ultra-long range:** Single molecules can be >300 kbp, far longer than any current sequencing read technology
- **Sequence-independent:** Captures structural information based on label positions, not sequence — useful for regions where sequence-based assembly fails (highly repetitive regions, segmental duplications)
- **Gap sizing:** Unlike Hi-C scaffolding (which uses arbitrary N lengths), optical maps can estimate the physical distance between two adjacent contigs, allowing sized gaps
- **Chimeric join detection:** Misassembled contigs that span two genomic regions can be detected when the in-silico map of the contig conflicts with the optical map

### Bionano CMAP Format

The optical map data is stored in CMAP format, which encodes the positions of fluorescent labels along each DNA molecule. This is the standard output format from the Bionano Saphyr instrument and Bionano Access software.

---

## Tool: Bionano Hybrid Scaffold

**Version:** 3.7.0+galaxy3  
The Bionano Hybrid Scaffold tool (part of the Bionano Solve suite) automates the integration of sequence assembly and optical map data.

### Five-Step Internal Workflow

1. **Generate in silico maps** from the sequence assembly by computationally digesting the contig sequences at the same restriction motif used in the wet-lab optical mapping experiment
2. **Align in silico maps** against the experimental Bionano optical maps, identifying conflicts (points where the sequence assembly disagrees with the physical map)
3. **Resolve conflicts** by cutting contigs at conflict positions (in VGP mode, both sequence and map conflicts result in cuts rather than one overriding the other)
4. **Merge non-conflicting maps** into hybrid scaffolds combining sequence contigs and optical map spacing information
5. **Generate output files** in AGP and FASTA format

---

## Input

| File | Format | Description |
|---|---|---|
| `Hap1 contigs FASTA` | FASTA | Haplotype 1 contigs from hifiasm |
| `bionano.cmap` | CMAP | Bionano optical map |

> For this tutorial, only Hap1 was scaffolded with Bionano. In a full VGP pipeline run, both haplotypes would typically be scaffolded.

---

## Parameters

| Parameter | Value | Rationale |
|---|---|---|
| NGS FASTA | `Hap1 contigs FASTA` | The sequence assembly to scaffold |
| BioNano CMAP | `bionano.cmap` | The optical map data |
| Configuration mode | VGP mode | Uses VGP-specific alignment scoring and conflict resolution settings |
| Genome maps conflict filter | Cut contig at conflict | When the optical map conflicts with the sequence assembly, cut the contig |
| Sequences conflict filter | Cut contig at conflict | When the sequence map conflicts with the optical map, cut the contig |

The "cut at conflict" strategy is conservative — it breaks contigs at points of disagreement rather than forcing a join. This avoids introducing misassemblies but may produce slightly more, shorter scaffolds at disputed regions.

---

## Output and Post-processing

The Bionano Hybrid Scaffold tool produces two key outputs:

| Output | Description |
|---|---|
| `NGScontigs scaffold NCBI trimmed` | Contigs that were successfully integrated into Bionano-supported scaffolds |
| `NGScontigs not scaffolded trimmed` | Contigs with no Bionano support — too small, no overlapping optical map coverage, or in conflict regions |

These two files were concatenated using **Concatenate multiple datasets** to produce the complete scaffolded assembly:

**Output:** `Hap1 assembly bionano`

This ensures no contigs are lost — those successfully scaffolded contribute their ordered/oriented sequence, while unscaffolded contigs are retained as-is.

---

## Post-scaffolding Evaluation

gfastats was run on `Hap1 assembly bionano` to measure the impact of scaffolding on assembly statistics.

| Parameter | Value |
|---|---|
| Input | `Hap1 assembly bionano` |
| Expected genome size | 11747160 |

Key metrics to observe after Bionano scaffolding:
- Reduction in the number of sequences (contigs merged into scaffolds)
- Increase in the length of the largest scaffold
- Improvement in N50 and NG50

---

## Notes

- **Sized gaps:** One of the advantages of Bionano scaffolding over Hi-C scaffolding is that gap lengths can be estimated from the physical distance between adjacent label positions on the optical map. This produces more biologically meaningful N's in the gap regions compared to the arbitrary 200 or 500 N's used by Hi-C scaffolders.
- **Configuration mode matters:** Using VGP mode ensures the alignment parameters are tuned for the specific coverage levels and label density typical of VGP-grade optical maps. Using default settings with VGP data may produce suboptimal alignments.
- **This step is optional in the full VGP pipeline:** If no Bionano data is available, the Hi-C scaffolding step (YaHS) can be applied directly to the hifiasm contigs. Bionano adds an intermediate level of scaffolding that improves the input quality for Hi-C scaffolding but is not strictly required.

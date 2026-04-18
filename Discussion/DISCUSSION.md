# Discussion

## Overview

The primary goal of this tutorial was to assemble a high-quality, chromosome-level genome of *Saccharomyces cerevisiae* S288C from synthetic PacBio HiFi reads, using a multi-technology pipeline combining HiFi long reads, Bionano optical maps, and Illumina Hi-C chromatin conformation data. Overall, the assembly was highly successful and the final result closely matches the known reference genome, validating both the data quality and the VGP pipeline methodology.

---

## Genome Profiling

Before assembly, GenomeScope2 estimated the haploid genome size at approximately **11.7 Mb** with a heterozygosity rate of **~0.576%** and a diploid sequencing coverage of **~50×**. The k-mer spectrum showed a clean bimodal distribution with peaks at ~25× (heterozygous) and ~50× (homozygous), consistent with a diploid organism sequenced at uniform depth. The model fit exceeded 93%, which gave us strong confidence in these estimates before a single contig was assembled.

The low heterozygosity (~0.576%) was notable. *S. cerevisiae* S288C is a well-characterized laboratory strain, so this result was expected — but it does mean the assembler had relatively little haplotype divergence to work with when phasing. In a wild-caught or more heterozygous organism, phasing would be substantially more challenging and the Hi-C data would be even more critical.

---

## Contig Assembly

Using hifiasm in Hi-C phased mode produced two haplotype assemblies (Hap1 and Hap2) with 16 and 17 contigs respectively and total lengths of ~11.3 Mb and ~12.2 Mb. Given that the *S. cerevisiae* S288C reference genome is ~12.1 Mb (16 chromosomes + mitochondrial DNA), these values are very close to expectation.

The slight difference in total length between Hap1 (~11.3 Mb) and Hap2 (~12.2 Mb) is worth noting. This asymmetry is common in diploid assemblies — one haplotype tends to capture slightly more sequence depending on how heterozygous regions are resolved during graph traversal. Neither value is alarming, and the combined span of both haplotypes covers the expected diploid content well.

The fact that hifiasm produced only 16–17 contigs at this stage — before any scaffolding — is an impressive result that reflects the power of HiFi reads. With older short-read technologies, producing thousands of fragmented contigs from this same genome would have been expected. The long read lengths (10–25 kbp) allow the assembler to span most repetitive regions, which is the primary bottleneck in eukaryotic genome assembly.

---

## Assembly Quality Metrics

BUSCO results confirmed high gene completeness for both haplotypes against the Saccharomycetes lineage database. The very low proportion of duplicated BUSCO genes in either haplotype assembly indicates that the Hi-C phasing worked effectively — if both haplotypes were collapsed into a single assembly, we would expect to see elevated BUSCO duplication levels.

Merqury's spectra-cn and spectra-asm plots supported this interpretation. The spectra-asm plot showed that homozygous k-mers (~50× coverage) were correctly shared between both haplotype assemblies, while haplotype-specific k-mers (~25× coverage) were appropriately split between Hap1 and Hap2. This is the expected pattern for a well-phased diploid assembly and confirms that Hi-C phasing in hifiasm distributed the haplotypes sensibly rather than mixing them.

The grey region of low-frequency k-mers (sequencing errors) was minimal, consistent with the high accuracy of HiFi reads. This contrasts with older long-read technologies (early Oxford Nanopore, pre-HiFi PacBio), where error correction would have been a major computational step before assembly.

---

## Scaffolding

### Bionano Optical Maps

Bionano hybrid scaffolding improved assembly contiguity by merging contigs separated by structural regions that sequence-based assembly alone could not resolve. The optical maps provide physical distance information independent of sequence similarity, which is especially useful for spanning large tandem repeats and segmental duplications that would otherwise leave assembly gaps.

### Hi-C Scaffolding with YaHS

The most dramatic improvement came from Hi-C scaffolding with YaHS. The pre-YaHS contact map showed 17 distinct scaffolds but with some ordering artifacts visible as off-diagonal signal. After YaHS, the contact map showed clean, well-separated triangular blocks along the diagonal — one per chromosome — with the vast majority of contact signal concentrated near the diagonal. This is the expected pattern for a correctly assembled genome, where loci on the same chromosome interact more frequently the closer they are in linear sequence.

The inversions visible when comparing the pre- and post-YaHS contact maps (highlighted in the tutorial) demonstrate the real utility of Hi-C scaffolding — it identified contigs that Bionano had placed in the wrong orientation and corrected them. This kind of error is essentially invisible to sequence-based QC but is immediately apparent in a contact map.

---

## Final Assembly vs. Reference

The final assembly closely matched the *S. cerevisiae* S288C reference genome across all key metrics:

| Metric | Our Assembly | Reference |
|---|---|---|
| Total length | ~12.1 Mb | ~12.1 Mb |
| Number of scaffolds | ~17 | 17 (16 chr + mtDNA) |
| Contact map topology | Clean diagonal blocks | Clean diagonal blocks |
| BUSCO completeness | High | — |

The near-identical Hi-C contact maps between our assembly and the reference genome are perhaps the most compelling evidence of assembly quality. It means not only that we assembled the right sequences, but that we ordered and oriented them correctly at the chromosome scale — which is the ultimate goal of a chromosome-level genome assembly.

---

## Limitations and Considerations

A few caveats are worth mentioning:

- **Synthetic reads:** The HiFi reads used in this tutorial were synthetically generated from the reference genome rather than produced from a real sequencing experiment. This means sequencing errors, coverage biases, and library preparation artifacts present in real data are absent here. A real genome assembly would likely require more troubleshooting, particularly around adapter contamination, coverage outliers, and chimeric reads.

- **Small, low-complexity genome:** *S. cerevisiae* S288C is a small (~12 Mb), relatively low-repeat genome compared to vertebrates. The VGP pipeline was designed primarily for large, complex vertebrate genomes where TE content, high heterozygosity, and polyploidy create far greater assembly challenges. The clean results here reflect an optimal-case scenario.

- **Purging was not required:** Because we used Hi-C phased mode in hifiasm, false duplications were already minimized by proper haplotype separation. In a solo assembly (HiFi only, no Hi-C phasing), the purging step with purge_dups would have been necessary to remove haplotigs from the primary assembly.

- **No decontamination step:** The tutorial workflow ends at scaffolding. A production-grade assembly would proceed to decontamination (Workflow 9 in the full VGP pipeline) to remove non-target sequences such as microbial contamination or organellar DNA from the nuclear assembly.

---

## Conclusion

This tutorial demonstrated that the VGP assembly pipeline, even when run manually tool-by-tool on Galaxy, is capable of producing a near-reference-quality chromosome-level assembly. The combination of HiFi long reads (for accurate contig assembly), Bionano optical maps (for intermediate-scale scaffolding), and Hi-C data (for chromosome-scale ordering and orientation) is a genuinely powerful and complementary set of technologies. Each layer addresses the limitations of the others, and the final result — 17 scaffolds matching the known 16 chromosomes plus mitochondrial DNA of *S. cerevisiae* — validates the pipeline end-to-end.

For future work on vertebrate genomes, the same pipeline would be expected to require substantially more compute time, careful QC at each stage, likely a purging step, and manual curation of the final Hi-C contact map to resolve any remaining misassemblies.

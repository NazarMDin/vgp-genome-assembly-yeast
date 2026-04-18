# Glossary of Key Terms

A reference for key terminology used throughout this genome assembly project.

---

## Assembly Terms

**Contig**  
A contiguous, gap-free sequence in an assembly — the basic unit produced by an assembler from sequencing reads.

**Scaffold**  
One or more contigs joined by stretches of unknown sequence (represented as N's). Scaffolds are generated using additional long-range data (optical maps, Hi-C) and represent a higher level of assembly than contigs.

**Unitig**  
The smallest unambiguous unit of an assembly graph. A unitig represents a path through the graph where there is no branching — every internal node has exactly one input and one output edge.

**Assembly graph**  
A graph representation of the genome inferred from sequencing reads, showing sequences (nodes) and their overlaps (edges). The starting point for all assemblers.

**GFA (Graphical Fragment Assembly)**  
A file format for representing assembly graphs, including sequences, nodes, and edges/overlaps. More information-rich than FASTA.

---

## Haplotype & Phasing Terms

**Haplotype**  
One copy of a chromosome in a diploid organism. Humans and most vertebrates have two copies (maternal and paternal) of each chromosome.

**Phasing**  
The process of assigning assembled contigs or sequences to one specific haplotype (e.g., maternal vs. paternal). Well-phased assemblies separate the two haplotypes rather than mixing them.

**Pseudohaplotype assembly**  
An assembly consisting of long phased blocks separated by regions where haplotypes cannot be distinguished. Represented by a primary + alternate assembly pair. Can suffer from "switch errors."

**Primary assembly**  
The more complete of the two assemblies in a pseudohaplotype pair. Contains both homozygous regions and one copy of heterozygous regions. Used for most downstream analyses.

**Alternate assembly**  
Contains only the alternate alleles (haplotigs) not represented in the primary assembly. Less complete than the primary.

**Haplotig**  
A contig representing one specific haplotype at a heterozygous region. Found in the alternate assembly.

**Switch error**  
An error in phased assemblies where the assembly switches between the two haplotypes mid-sequence, rather than staying on one haplotype throughout.

---

## Sequencing Technology Terms

**HiFi reads (PacBio)**  
Long reads (10–25 kbp) produced by PacBio's SMRT sequencing with ≥99% accuracy (Q20). Combines the length advantage of long-read sequencing with the accuracy typical of short-read sequencing.

**Ultra-long reads**  
Reads >100 kbp, typically from Oxford Nanopore Technology. Lower accuracy than HiFi but can span very complex repeat regions.

**Hi-C**  
A sequencing method that captures the 3D organization of chromatin in the nucleus. Loci that are physically close in the nucleus are sequenced together, providing long-range linking information used for scaffolding.

**Optical mapping (Bionano)**  
A technology that labels long DNA molecules at specific sequence motifs and images them, creating a physical restriction map of the genome. Provides very long-range structural information independent of sequencing.

**SMRT sequencing**  
Single-Molecule Real-Time sequencing — PacBio's long-read sequencing technology. HiFi reads are produced using circular consensus sequencing (CCS) of SMRT reads.

---

## Assembly Quality Terms

**N50**  
The contig/scaffold length at which ≥50% of the total assembly length is contained in contigs/scaffolds of that size or longer. A higher N50 indicates better contiguity.

**NG50**  
Like N50, but calculated relative to the reference genome size rather than the assembly size. More meaningful for comparing assemblies of the same species.

**BUSCO**  
Benchmarking Universal Single-Copy Orthologs. A method for assessing assembly completeness by checking for the presence of genes expected to be in single copy in a given clade. Missing BUSCO genes may indicate incomplete assembly.

**QV (Quality Value)**  
A Phred-scale measure of assembly base accuracy. QV30 = 99.9% accuracy, QV40 = 99.99%, QV50 = 99.999%. Calculated by Merqury using k-mer comparisons.

**k-mer**  
A sequence of exactly k nucleotides. k-mer counting reveals the frequency distribution of all possible k-length substrings in the reads, which reflects genome properties.

**Merqury**  
A reference-free assembly evaluation tool that uses k-mers from the raw reads to assess completeness and phasing quality of assemblies.

---

## Assembly Error Terms

**False duplication**  
An assembly error where a genomic region is represented twice as separate sequences. Can be caused by haplotypic duplication or unresolved overlaps.

**Haplotypic duplication**  
A type of false duplication where two haplotypes of a heterozygous region are both placed in the same assembly instead of being separated into two haplotype assemblies.

**Misassembly**  
A structural error in genome reconstruction — sequences that are not adjacent in the real genome being placed next to each other in the assembly.

**Missed join**  
The opposite of a misassembly — two sequences that are adjacent in the real genome but are not connected in the assembly. Identified and corrected during manual curation using Hi-C data.

**Purging**  
The process of removing false duplications, collapsed repeats, and very-low-coverage regions from an assembly. Haplotigs removed from the primary assembly are typically added to the alternate assembly.

---

## Genome Complexity Terms

**Heterozygosity**  
The proportion of sites in the genome where the two haplotypes differ. High heterozygosity makes phasing harder and can cause assemblers to collapse or duplicate heterozygous regions.

**Transposable elements (TEs)**  
Repetitive sequences that can occur at multiple locations throughout a genome ("interspersed repeats"). High TE content is strongly correlated with genome size and assembly difficulty.

**Tandem repeats (TRs)**  
Repetitive sequences that appear in tandem at a single locus. Problematic for assembly when repeat unit length approaches read length.

**Diploid**  
Having two copies of each chromosome (2n). Most vertebrates are diploid.

**Ploidy**  
The number of complete sets of chromosomes in a cell. Diploid = 2, triploid = 3, tetraploid = 4, etc.

---

## Scaffolding Terms

**Contact map**  
A visual representation of Hi-C interaction frequencies between all pairs of genomic loci. Used to evaluate scaffolding quality — correctly assembled chromosomes appear as distinct triangular blocks along the diagonal.

**YaHS**  
Yet Another Hi-C Scaffolding tool. Uses Hi-C contact information to order and orient contigs into chromosome-scale scaffolds without requiring a pre-specified chromosome number.

**T2T (Telomere-to-Telomere)**  
A completely gapless chromosome assembly from one telomere to the other. The gold standard for genome assembly quality.

**Manual curation**  
The process of manually reviewing and correcting an assembly based on raw supporting evidence (Hi-C contact maps, coverage tracks, repeat annotations) to resolve misassemblies and missed joins.

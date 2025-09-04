---
layout: post
title:  "Cross-sample homopolymer polishing with Pypolca"
date:   2025-09-04
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.17051700.svg)](https://doi.org/10.5281/zenodo.17051700)


## _Cryptosporidium_ assembly

I was recently working with [Torsten Seemann](https://tseemann.github.io) and our colleagues at the [Centre for Pathogen Genomics](https://biomedicalsciences.unimelb.edu.au/departments/microbiology-Immunology/research/centre-for-pathogen-genomics) to assemble a _Cryptosporidium hominis_ genome from ONT reads. It's small for a eukaryote at only 9.2 Mbp, has low GC content (~30%) and, importantly for this post, has lots of long `A`/`T` homopolymers (~100 of them ≥20 bp). The ONT sequencing was deep, and the assembly went smoothly: T2T for all eight chromosomes.[^autocycler] However, I expect it contains homopolymer-length errors – while R10.4.1 is better than R9.4.1, I'm still suspicious of homopolymers >10 bp.

To illustrate the problem, I took a long homopolymer from the _Crypto_ genome and used [squigulator](https://github.com/hasindu2008/squigulator) to simulate the raw ONT signal. For long stretches of a single base, the signal flatlines, making it difficult for the [basecaller](https://github.com/nanoporetech/dorado) to determine the precise homopolymer length:

<p align="center"><picture><source srcset="/assets/images/homopolymer_squiggle-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/homopolymer_squiggle.png" alt="Long homopolymer ONT squiggle" width="100%"></picture></p>

Normally, I'd fix homopolymer lengths (and any other lingering errors) with short reads from the same isolate using [Polypolish](https://github.com/rrwick/Polypolish) and [Pypolca](https://github.com/gbouras13/pypolca). But we didn't have short reads for this genome, so I wondered: could I correct homopolymers using short reads from a closely related isolate instead?[^existing] I'll call this _cross-sample homopolymer polishing_.



## Homopolymer-only polishing

So I decided to add a homopolymer-only polishing feature to Pypolca. When used, Pypolca will ignore all changes except for length adjustments in homopolymers above a threshold. George Bouras kindly merged [my pull request](https://github.com/gbouras13/pypolca/pull/29) and released it in [v0.4.0](https://github.com/gbouras13/pypolca/releases/tag/v0.4.0), so this feature is now available. Documentation is [here](https://github.com/gbouras13/pypolca#homopolymer-only-mode).

To try it, I downloaded all [NCBI assemblies of _Cryptosporidium_](https://www.ncbi.nlm.nih.gov/datasets/genome/?taxon=5806) and identified the [closest relative](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_001483515.1) to my genome using [Mash](https://github.com/marbl/Mash). I then downloaded [Illumina reads](https://www.ncbi.nlm.nih.gov/sra/SRR1557959) for that sample[^readspaper] and polished with:
```
pypolca run -a draft.fasta -1 SRR1557959_1.fastq.gz -2 SRR1557959_2.fastq.gz -t 16 --homopolymers 6
```

This changed the length of 343 homopolymers:[^defaults]

<p align="center"><picture><source srcset="/assets/images/pypolca_homopolymers-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/pypolca_homopolymers.png" alt="Pypolca homopolymer-only polishing distributions" width="90%"></picture></p>

The top plot shows the distribution of all homopolymers in the genome.[^random] The bottom plot shows homopolymers whose length changed. Both plots use a pseudo-log y axis. The dashed line marks the `--homopolymers 6`{:.nowrap} threshold, i.e. shorter homopolymers were not allowed to change.

The results match my expectations: very few shorter homopolymers were altered (e.g. 2/17297 6-mers and 3/7070 7-mers) where ONT is reliable, but about one-quarter of longer homopolymers changed (e.g. 6/26 19-mers and 9/30 20-mers) where ONT struggles.

I also annotated the genome before and after homopolymer-only polishing with [Companion](https://companion.gla.ac.uk), and it seemed to improve the annotation: gene count increased (4069 → 4076), pseudogene count decreased (41 → 36) and the fraction of bases annotated increased (76.968% → 76.979%).



## Discussion

Cross-sample homopolymer polishing looks promising, but it depends on several assumptions:
1. ONT-only assemblies have length errors in long homopolymers.
2. Illumina reads handle long homopolymers better than ONT reads.
3. Closely related genomes usually share the same homopolymer lengths.
4. Homopolymer lengths are biologically consistent (i.e. little true variation).

I ordered these by my confidence. Assumption 1 seems safe. I think assumption 2 is usually true, though short reads can struggle in extreme GC contexts.[^gc] I'm less sure about assumptions 3 and 4, which probably depend on the organism – what holds in _Crypto_ may not in other taxa.

So while Pypolca now supports cross-sample homopolymer-only polishing, treat the biological validity as experimental. If the assumptions hold, it may work very well. For our _Crypto_ genome... I'm not yet sure. We'll be sequencing our isolate with Illumina to get a firmer answer.




## Footnotes

[^autocycler]: This was my first time trying [Autocycler](https://github.com/rrwick/Autocycler) on a eukaryote genome. It worked well, except for the telomeres at contig ends, which I had to manually repair. See the Autocycler graph [here](https://github.com/rrwick/Autocycler/wiki/FAQ-and-miscellaneous-tips#can-autocycler-be-used-on-eukaryote-genomes).

[^existing]: I couldn't find an existing tool that does exactly this. [This anvi'o script](https://anvio.org/help/main/programs/anvi-script-fix-homopolymer-indels) is similar, but it uses a reference genome (not reads) to set homopolymer lengths. [Homopolish](https://github.com/ythuang0522/homopolish) is also similar, but it can make non-homopolymer changes (see [this issue](https://github.com/ythuang0522/homopolish/issues/49)).

[^readspaper]: The source of these reads: [Comparative genomic analysis reveals occurrence of genetic recombination in virulent _Cryptosporidium hominis_ subtypes and telomeric gene duplications in _Cryptosporidium parvum_](https://doi.org/10.1186/s12864-015-1517-1).

[^defaults]: For comparison, Pypolca with default settings (not homopolymer-only) made >1600 changes, most of which were not in homopolymers. Some may have been genuine fixes, but I suspect most are biological differences between our genome and the downloaded reads.

[^random]: These homopolymers are much longer than I'm used to seeing in bacteria! As a comparison, I generated random sequences with the same length and GC content, and their longest homopolymer was typically around 15-16 bp.

[^gc]: Because homopolymers are pure `A`/`T` or `G`/`C`, they skew local GC. In _Crypto_, all long homopolymers are `A`/`T`, which combined with an already low average GC (~30%), means they often fall in very low-GC regions (<20%). Some Illumina preps (e.g. Nextera XT) do not do well with this.

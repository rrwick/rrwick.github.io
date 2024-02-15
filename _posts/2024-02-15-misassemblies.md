---
layout: post
title:  "A tale of two misassemblies"
date:   2024-02-15
author: Ryan Wick, George Bouras
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10662704.svg)](https://doi.org/10.5281/zenodo.10662704)


I've been getting a lot of mileage out of the nine [ATCC](https://www.atcc.org) genomes featured in some recent blog posts[^recent]. Currently, George Bouras and I are using them in a paper which examines low-depth Illumina polishing – stay tuned for that! While working on this paper, however, we saw two interesting misassemblies that I describe here. Both cases occurred with ONT-only assemblies ([sup@v4.3.0 basecalling](https://github.com/nanoporetech/dorado#dna-models)) performed by [Flye](https://github.com/fenderglass/Flye).



## Misassembly 1: _Vibrio cholerae_

The first misassembly happened with [_Vibrio cholerae_ ATCC-14035](https://www.atcc.org/products/14035) ([reads here](https://trace.ncbi.nlm.nih.gov/Traces/?run=SRR27638401)). This genome has two circular chromosomes, the larger of which has an unusually long repeat (by bacterial standards) at 34 kbp.[^phage] There are two exact copies of this repeat on opposite strands with 43 kbp of sequence in between. This configuration means that without any long reads to entirely span the repeat, the orientation of that middle 43 kbp would be unclear – it could point in either direction. Here is an illustration of the two configurations (not to scale):

<p align="center"><picture><source srcset="/assets/images/misassemblies_1-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/misassemblies_1.png" alt="Two possible configurations around inverted repeat" width="70%"></picture></p>

Thankfully, 10 reads do span the entire repeat, but they aren't consistent: eight support configuration A and two support configuration B.[^duplex] It seems that a mixture of both configurations was in the sample. Perhaps homologous recombination occurred in the repeat, flipping the middle sequence around.

In cases like this, I would like an assembler to recognise the presence of heterogeneity and produce a contig consistent with the majority of the reads (configuration A).[^annotate] However, heterogeneity can confuse assemblers, and Flye chose to put both configurations in its contig, like this:

<p align="center"><picture><source srcset="/assets/images/misassemblies_2-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/misassemblies_2.png" alt="Flye's misassembled contig" width="100%"></picture></p>

This is a linear contig with configuration A at the end and configuration B at the start. I can understand why Flye did this – it's a single contig compatible with all of the reads. But it's not an ideal outcome, as this contig includes extra sequence[^extra] and has failed to circularise.



## Misassembly 2: _Listeria welshimeri_

The second misassembly happened with [_Listeria welshimeri_ ATCC-35897](https://www.atcc.org/products/35897) ([reads here](https://trace.ncbi.nlm.nih.gov/Traces/?run=SRR27638395)), where Flye deleted a 3956 bp chunk of the genome. George first noticed it when running [hybracter](https://github.com/gbouras13/hybracter) (which uses Flye), and he saw that the deletion occurred about 70% of the time. I tried to replicate the problem on my computer, and I found it to depend on thread count: using 1 thread or 5+ threads resulted in a correct contig (no deletion) while using 2–4 threads resulted in the deletion.[^threads]

I have less to say about this misassembly because I cannot figure out why it happened. The deleted piece of the genome is not in a repetitive region (where misassemblies often occur), and I can't spot any heterogeneity – all of the reads from this locus support the correct sequence and none contain the deletion. Very mysterious! It might require a deep dive into Flye's algorithm to figure out what's going on here.



## Conclusions

This post is not meant to be a criticism of Flye. I quite like Flye – it's one of my favourite long-read assemblers![^favourites] But all assemblers, Flye included, make mistakes sometimes. So I wanted to remind readers that misassemblies can and do occur. Assemblies, especially those made with a single tool, should be viewed with some scepticism.

Structural heterogeneity is often the cause of misassemblies, because assemblers get confused with mixtures of different genome structures. If you were expecting a circular contig[^circularity] (i.e. the genome is circular) but got a linear contig, that is a clue that a misassembly due to structural heterogeneity may have occurred. But sometimes misassemblies have no obvious cause – they are simply a mistake made by the assembler.

Another interesting lesson is that assemblers can be non-deterministic: given the same input data, different runs may produce slightly different contigs.[^skesa] For some assemblers (including Flye), thread count can be a factor. In many scenarios, this non-determinism isn't a problem – just don't assume that re-running an assembly will produce the exact same result.

Finally, I'll use this opportunity to once again plug my tool [Trycycler](https://github.com/rrwick/Trycycler/wiki). It is the best way I know of to avoid misassemblies like the two described in this post. Trycycler takes as input multiple separate assemblies of the same genome (ideally produced from different subsets of reads), and it combines them into a consensus assembly. Misassemblies can be caught at the [reconciliation step](https://github.com/rrwick/Trycycler/wiki/Reconciling-contigs) (where they often create problems with circularisation) or at the [consensus step](https://github.com/rrwick/Trycycler/wiki/Generating-a-consensus) (where only the majority variant at each locus is used). The caveat is that Trycycler usually takes some manual work, so it's a slow way assemble a genome.



## Footnotes

[^recent]: I first used these genomes [here](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html) to look at accuracy after the move to 5 kHz sampling, and then again [here](https://rrwick.github.io/2023/12/18/ont-only-accuracy-update.html) after new Dorado basecalling models were released.

[^phage]: George confirmed using [pharokka](https://github.com/gbouras13/pharokka) that the repeat is prophage.

[^duplex]: Actually, these two reads are a duplex pair, i.e. they came from the two strands of a single piece of DNA. So there was only one sequenced DNA fragment supporting configuration B.

[^annotate]: Even better would be to include the most common configuration in the assembly but also add an annotation to describe the heterogeneity, but I'm not aware of any assembler that can do this.

[^extra]: The correct counts for this genome are 2× repeat and 1× middle. The misassembled linear contig contains ~2.5× repeat and ~1.5× middle.

[^threads]: The problem isn't always due to thread count – George consistently used 4 threads and he sometimes got the deletion, sometimes did not.

[^favourites]: My other favourite is probably [Canu](https://github.com/marbl/canu), especially since I wrote [this script](https://github.com/rrwick/Trycycler/blob/main/scripts/canu_trim.py) to trim off start-end overlap in Canu's circular contigs. Both Flye and Canu usually produce reliable assemblies, but Flye is faster so I use it the most.

[^circularity]: Different assemblers have different ways of indicating whether a contig is circular or linear. Some, like Flye, produce a [GFA](https://gfa-spec.github.io/GFA-spec/GFA1.html) assembly graph that you can view in [Bandage](https://github.com/rrwick/Bandage) to see circularisation. Canu includes `suggestCircular=yes` or `suggestCircular=no` in contig header lines. [Miniasm](https://github.com/lh3/miniasm) contig names end in `c` (for circular) or `l` (for linear).

[^skesa]: For short-read assemblies, [SKESA](https://github.com/ncbi/SKESA) is a tool which addresses this problem. It was specifically designed to be deterministic, regardless of thread count and read order.

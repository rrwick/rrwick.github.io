---
layout: post
title:  "A perfect bacterial genome from PacBio HiFi reads"
date:   2023-03-07
author: Ryan Wick
---




## Introduction

Some of my [recent research](https://doi.org/10.1371/journal.pcbi.1010905) is about getting a perfect (zero-error) bacterial genome assembly using a combination of Oxford Nanopore and Illumina reads. But I'm often asked about PacBio reads: can they deliver a perfect assembly on their own, i.e. without Illumina reads to polish?

I've spent a lot of time using Oxford Nanopore sequencing and am comparatively ignorant when it comes to PacBio. I had some experience with PacBio RS II reads in 2017, assembling with HGAP and polishing with Quiver, but a lot has changed since then. Long-read assemblers are now much better (e.g. [Canu](https://github.com/marbl/canu) and [Flye](https://github.com/fenderglass/Flye)), and the noisy single-pass PacBio CLR reads have mostly been replaced by accurate multi-pass [HiFi reads](https://www.pacb.com/technology/hifi-sequencing).

So I looked for a publicly available dataset that would let me experiment with PacBio reads, and I found this paper: [Comparison of long-read sequencing technologies in interrogating bacteria and fly genomes](https://academic.oup.com/g3journal/article/11/6/jkab083/6188627). The authors sequenced an _E. coli_ E2348/69 genome using Illumina, ONT, PacBio CLR and PacBio HiFi. The read sets all came from the same DNA (so they are consistent) and they are _very_ deep[^1]. So I downloaded the reads, made some assemblies and counted the errors!



## Ground-truth assembly and heterogeneity

I first wanted a perfect assembly of this _E. coli_ E2348/69 genome that I could use as a ground truth. While the authors [provided an assembly on NCBI](https://ncbi.nlm.nih.gov/assembly/GCF_014117345.2), it was made using Unicycler, which I know can make mistakes (see Figure S4 of the [Trycycler paper](https://doi.org/10.1186/s13059-021-02483-z)). Briefly, I did Trycycler+Medaka+Polypolish+POLCA assemblies using multiple different long-read sets (PacBio CLR, PacBio HiFi, ONT) and then manually investigated any differences between them[^2].

However, my first attempts ran into problems[^3] at two loci. In both, a piece of sequence (~1.9 kbp in the first, ~1.5 kbp in the second) was present in a heterogeneous mixture of two orientations: ~55% of the time the forward sequence was present, and ~45% of the time the reverse complement sequence was present. I'm not sure what the biology is here, but both loci were annotated as 'tail fiber assembly protein'. They occurred far apart in the genome, so I don't know if the two loci were correlated.

Near-50:50 heterogeneous mixtures like this can cause problems with assembly and polishing, so I considered the majority (~55%) variants to be 'correct' and discarded long reads which supported the minority (~45%) variants. To ensure even depth over the whole genome, I also discarded a random 45% of the reads that didn't align to these regions[^4]. Since the read sets were still quite deep, I then used [Filtlong](https://github.com/rrwick/Filtlong) to cut them down to 2.5 Gbp (about 500× depth)[^5].

After this culling of heterogeneity, my assemblies were very clean and in good agreement. The HiFi+Illumina and CLR+Illumina hybrid assemblies were exactly the same. I also tried an ONT+Illumina hybrid assembly, but this paper's ONT reads weren't ideal (R9.4.1, basecalled with Guppy v4.2.2 high-accuracy model)[^6]. There were 128 differences between the ONT+Illumina assembly and the PacBio+Illumina assemblies, but they were all in homopolymers or methylation motifs, regions [known to cause problems with older ONT reads](https://doi.org/10.1186/s13059-019-1727-y), so they didn't worry me. I was therefore confident in using my PacBio+Illumina hybrid assembly as a perfect ground truth genome sequence.

It's also worth noting that this genome contains a [small plasmid](https://ncbi.nlm.nih.gov/nuccore/NZ_CP059842.2). While this plasmid is present in the Illumina reads, I've essentially ignored it in this experiment because it's almost entirely absent from the PacBio read sets[^7].



## Can PacBio-only deliver a perfect assembly?

Now that I had a ground truth, I could assess a few different assemblies using both CLR and HiFi reads. In addition to the Trycycler assemblies I already made, I tried [Canu](https://github.com/marbl/canu) and [Flye](https://github.com/fenderglass/Flye) on their own[^8]. All assemblies were done using my heterogeneity-culled post-Filtlong read sets.

For both HiFi and CLR read sets, Trycycler produced a perfect assembly with zero errors! So even though CLR reads have far more errors (~90% read identity for CLR vs >99.9% for HiFi) they can still produce a perfect assembly. I.e. the errors in PacBio CLR reads are sufficiently random to average out in the consensus.

The Canu assembly of the HiFi reads also produced a perfect assembly[^9]. However, the Canu assembly of the CLR reads contained 11 errors, all in homopolymers. So even though CLR reads can theoretically produce a perfect assembly, Canu couldn't quite pull this off.

Flye also produced a perfect assembly with HiFi reads but not with CLR reads. The Flye CLR assembly contained 36 errors, most of which were in homopolymers. So Flye also failed to produce a perfect CLR assembly, even though it was possible.



## Discussion and conclusions

This little experiment only used one genome, so I can't draw broad conclusions. But it demonstrates that at least in some cases, HiFi reads can easily (no Trycycler, no polishing) produce a perfect bacterial genome assembly.

This wasn't quite possible for CLR reads, but that may not be relevant anymore – CLR reads are mostly a thing of the past. In addition to being easier to assemble, HiFi reads are easier to work with than CLR reads: alignment is faster, alignment visualisation (e.g. in [IGV](https://software.broadinstitute.org/software/igv)) looks nicer and post-assembly polishing isn't necessary so you don't need to keep the raw files[^10].

But despite this success, I still have some reservations about HiFi-only bacterial assembly:
* Size-selection of PacBio libraries can exclude small plasmids (as was the case for this genome).
* The HiFi reads I used were about 12-15 kbp, which is long enough for most but not all bacterial genomes. For genomes with particularly long repeats (e.g. >30 kbp), a high-N50 ONT sequencing run may be necessary to get a complete assembly.
* The read sets I used were very deep, so accuracy might suffer with a more typical read depth (e.g. ~100×).
* The longest homopolymer in the _E. coli_ E2348/69 genome was 10 bp[^11], so longer homopolymers may still be a problem[^12].

My fantasy money-is-no-object experiment would be to take 100+ diverse bacterial genomes and sequence each with deep HiFi, ONT and Illumina. This would allow for a good comparison of different approaches (e.g HiFi-only vs ONT+Illumina hybrid) at various sequencing depths.



## Footnotes

[^1]: The HiFi reads were >4000× deep and the CLR reads were >12000× deep – see [Table 1](https://academic.oup.com/view-large/304735460).

[^2]: See my [perfect bacterial genome paper]((https://preprints.scielo.org/index.php/scielo/preprint/view/5053/version/5357)) and its [accompanying tutorial](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki) for more details on the methods I use.

[^3]: If all goes well in a Trycycler assembly, the remaining errors (e.g. homopolymer-length errors) should be evenly distributed around the genome. So when polishers (e.g. Medaka or Polypolish) make a lot of changes in small region, that's a sign that something is wrong.

[^4]: Throwing out nearly half the reads might seem wasteful, but the read sets were so deep that I could get away with it here.

[^5]: The PacBio CLR reads didn't have Q-scores (the FASTQs just had `!` for every base) while the HiFi reads did. This means that Filtlong subsetting of the CLR reads was only based on length, but Filtlong subsetting of the HiFi reads was based on both length and quality.

[^6]: I would have re-basecalled using Guppy v6 super-accuracy model to get more up-to-date ONT reads, but the raw files necessary for basecalling weren't available.

[^7]: This is due to the size-selection of the PacBio library – the reads are longer than the small plasmid.

[^8]: There are [other long-read assemblers](https://f1000research.com/articles/8-2138), some of which are quite good. But I limited this test to Canu and Flye because they are popular, are easy to run and have explicit settings for both CLR and HiFi reads.

[^9]: Canu assemblies of circular sequences can have a lot of start-end overlap, but the FASTA header contains the necessary info to trim it off. I used Trycycler's [`canu_trim.py`](https://github.com/rrwick/Trycycler/blob/main/scripts/canu_trim.py) script to produce overlap-free Canu assemblies.

[^10]: While CLR assemblies can be improved (mainly homopolymers) by polishing with Arrow, HiFi reads are generated using an Arrow-like algorithm, so the raw-signal polishing takes place at the read-generation step not the assembly step.

[^11]: There are six 10-bp homopolymers in the genome: four `A/T` and two `C/G`.

[^12]: To be clear, I'm only concerned with _consensus_ homopolymer accuracy, not read-level homopolymer accuracy. E.g. I don't mind if individual reads occasionally get homopolymers wrong, as long as the error is sufficiently unbiased to produce the correct sequence in the consensus.

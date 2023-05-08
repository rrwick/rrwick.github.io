---
layout: post
title:  "ONT-only accuracy with R10.4.1"
date:   2023-05-05
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7898220.svg)](https://doi.org/10.5281/zenodo.7898220)


Since publishing our 'perfect assembly' [paper](https://doi.org/10.1371/journal.pcbi.1010905) and accompanying [online tutorial](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki), I've had a chance to apply the method to a few more bacterial genomes using R10.4.1 ONT reads. I was especially interested in determining the accuracy of R10.4.1-only bacterial genome assemblies. If you share my curiosity, read on!



## Methods

I used the following [ATCC](https://www.atcc.org/) genomes, sequenced by my colleague and co-author [Louise](https://twitter.com/JuddLmj), as test cases:
* [ATCC-10708 _Salmonella enterica_](https://www.atcc.org/products/10708)
* [ATCC-17802 _Vibrio parahaemolyticus_](https://www.atcc.org/products/17802)
* [ATCC-25922 _Escherichia coli_](https://www.atcc.org/products/25922)
* [ATCC-33560 _Campylobacter jejuni_](https://www.atcc.org/products/33560)
* [ATCC-BAA-679 _Listeria monocytogenes_](https://www.atcc.org/products/baa-679)

Each genome had deep ONT and Illumina reads, making them suitable candidates for long-read-first hybrid assembly.

To make the best possible ground-truth assembly for each genome, I used [Trycycler](https://github.com/rrwick/Trycycler/wiki), [Medaka](https://github.com/nanoporetech/medaka), [Polypolish](https://github.com/rrwick/Polypolish) and [POLCA](https://github.com/alekseyzimin/masurca), with lots of manual curation and checking throughout the process. Although I didn't have separate, independent read sets to verify the perfection of the assemblies, I encountered no significant difficulties, which bolsters my confidence in the results.

To quantify accuracy, I compared ONT-only assemblies of each genome to my ground-truth assembly, counting the number of differences.[^1] I tried the following ONT-only assembly methods: [Flye](https://github.com/fenderglass/Flye), [Canu](https://github.com/marbl/canu), Trycycler and Trycycler+Medaka. I chose Flye and Canu because (in my experience) they produce the best single-assembler assemblies.



## Results

This table shows the error count (top) and qscore (bottom) for each assembly:

| Genome                    | Flye          | Canu          | Trycycler    | Trycycler<br>+Medaka | Main cause of errors |
|---------------------------|:-------------:|:-------------:|:------------:|:--------------------:|----------------------|
| _Salmonella enterica_     | 52<br>Q49.7   | 35<br>Q51.4   | 13<br>Q55.7  | 9<br>Q57.3           | homopolymers         |
| _Vibrio parahaemolyticus_ | 149<br>Q45.4  | 91<br>Q47.5   | 52<br>Q50.0  | 81<br>Q48.0          | unknown methylation? |
| _Escherichia coli_        | 332<br>Q42.0  | 223<br>Q43.7  | 171<br>Q44.8 | 171<br>Q44.8         | M1.EcoMI methylation |
| _Campylobacter jejuni_    | 1004<br>Q32.5 | 1113<br>Q32.0 | 508<br>Q35.4 | 578<br>Q34.9         | CtsM methylation     |
| _Listeria monocytogenes_  | 12<br>Q53.9   | 7<br>Q56.2    | 0<br>Q∞      | 0<br>Q∞              | n/a                  |



## Discussion and conclusions

The first thing I noticed was how much the different genomes varied in accuracy. The Trycycler _Listeria_ genome had no errors, so I'll tentatively call this my very first perfect ONT-only assembly![^2] Conversely, the _Campylobacter_ genome contained numerous errors.[^3] ONT claims their platform can produce Q50 bacterial genomes, and while my median Trycycler accuracy was indeed ~Q50, there is a lot of variation.

In the more error-prone genomes, there were distinct patterns around the error locations. Most errors in the _E. coli_ genome occurred in `GAGACC`, an [M1.EcoMI methylation motif](https://doi.org/10.1128/mBio.01602-15). Most _Campylobacter_ errors were found in `RAATTY`, a [CtsM methylation motif](https://doi.org/10.1073%2Fpnas.1703331114). And most _Vibrio_ errors occurred at `TCG` sequences – I suspect this might also be related to a methylation motif, though I couldn't find any publications describing it.

I believe there are two main factors that determine ONT-only assembly accuracy. The first is the well-known issue of homopolymers.[^4] The second is DNA modifications: if a genome has an unusual modification, it can lead to systematic read errors and ultimately assembly errors. By 'unusual' I mean a modification and motif that were not included in the basecaller's training set.[^5] A robust solution to this problem would be to ensure that the basecaller's training set includes modified bases in all possible motifs.[^6] An easier species-specific solution would be to fine-tune the basecalling model, e.g. a _Campylobacter_ researcher could produce a _Campylobacter_-tuned model to avoid the `RAATTY` errors I encountered.[^7]

Regarding assembly methods, I would rank them in this order (best to worst): Trycycler, Canu and Flye. Canu usually produced a slightly better assembly than Flye, though it was slower.[^8] Both Canu and Flye could often make a completed genome on their own, but a bit of manual work was sometimes necessary.[^9] Trycycler always produced better assemblies than either Flye or Canu, but it's a more complicated process which involves human judgement and intervention.

The final thing I noticed is that Medaka often made the assembles worse! This contrasts with my previous experience (mostly with R9.4.1 reads) where Medaka almost always improved accuracy. While I'm not quite ready to abandon Medaka completely, I now assess assembly quality before and after running it, proceeding with whichever version looks better.[^10] I might also look more into [polishing with Clair3](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki/Polishing-with-Clair3) to see if it can consistently outperform Medaka.



## Footnotes

[^1]: I used my [`compare_assemblies.py`](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki/Comparing-assemblies) script to get a difference count.

[^2]: The _Listeria_ genome is tentatively perfect because Illumina-read polishing made zero changes – my final assembly was identical to my Trycycler assembly. If I wanted to be certain it was perfect, I would sequence it again (ideally on a different platform like PacBio) and verify that assembling the new reads produces the exact same sequence.

[^3]: A Q35 assembly is not very good – something I'd expect with older R9.4.1 reads, not with the newer R10.4.1 reads we used here. Q35 equates to approximately one error per 3200 bp, so a decent proportion of the genes in the genome will contain an error. 

[^4]: Homopolymer errors have decreased with the move from R9.4.1 to R10.4.1, but very long homopolymers (e.g. the _Salmonella_ genome had a C×21) can still cause problems.

[^5]: For example, in [a paper from 2019](https://doi.org/10.1186/s13059-019-1727-y), we found that [Dcm methylation](https://doi.org/10.1128%2Fecosalplus.ESP-0003-2013) caused assembly errors, but that no longer seems to be the case, presumably because native DNA with that modification/motif is now in the training sets.

[^6]: This could be difficult! The DNA for the training data would need to be made synthetically to include modifications in all motifs. And expanding the DNA alphabet (e.g. to 8 bases by adding 4mC, 5mC, 5hmC and 6mA) greatly increases the number of possible _k_-mers.

[^7]: This may be possible with [Bonito](https://github.com/nanoporetech/bonito), though I haven't tried myself.

[^8]: The Flye assemblies took about 15-20 minutes each, while Canu assemblies took about 1-2 hours. Each job got 32 CPUs.

[^9]: The _Salmonella_ Canu assembly contained an extra spurious contig (duplicated part of the chromosome) that I had to remove. I needed to circularise the large plasmid and manually add the smallest plasmid in the _E. coli_ Flye assembly. The _E. coli_ Canu assembly doubled the large plasmid in a single contig (97 kbp instead of 48.5 kbp). The _Listeria_ Flye and Canu assemblies both had an extra contig that I needed to remove.

[^10]: I use [ALE](https://github.com/sc932/ALE) for this, [read more here](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki/Assessing-quality-without-a-reference).

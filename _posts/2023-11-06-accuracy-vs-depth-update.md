---
layout: post
title:  "ONT accuracy vs depth update"
date:   2023-11-06
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10073814.svg)](https://doi.org/10.5281/zenodo.10073814)


After my [last post](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html), a common question arose (I'm paraphrasing):
> Those accuracy values are interesting, but they came from very-high-depth Trycycler assemblies. What sort of accuracy could I expect for a normal-read-depth Flye assembly?

It's a good question which I hope to address here. Consider this something of a sequel to my [2021 accuracy-vs-depth analysis](https://rrwick.github.io/2021/08/10/accuracy-vs-depth.html).





## Methods

Instead of using all nine genomes from my last post, I used just one genome per genus[^onepergenus]:
* _Campylobacter jejuni_ ([ATCC-33560](https://www.atcc.org/products/33560))
* _Escherichia coli_ ([ATCC-25922](https://www.atcc.org/products/25922))
* _Listeria monocytogenes_ ([ATCC-BAA-679](https://www.atcc.org/products/baa-679))
* _Salmonella enterica_ ([ATCC-10708](https://www.atcc.org/products/10708))
* _Vibrio parahaemolyticus_ ([ATCC-17802](https://www.atcc.org/products/17802))

And I only used the sup-basecalled reads (`dna_r10.4.1_e8.2_400bps_sup@v4.2.0`) from my last post for this analysis. While the fine-tuned bacterial research model did a little bit better, I thought I should stick to a standard [Dorado](https://github.com/nanoporetech/dorado) model to make these results more relevant.

For each genome, I used the [Trycycler-partitioned](https://github.com/rrwick/Trycycler/wiki/Partitioning-reads) reads for the largest replicon[^largest], which simplified the assembly and analysis: each read set should cleanly assemble into one circular contig. I then used [seqtk](https://github.com/lh3/seqtk) to produce 200 random subsamples for each read set with depths ranging from 0× to up to 400× (uniformly spaced on a square-root scale).[^maxdepth] I assembled each with [Flye](https://github.com/fenderglass/Flye) v2.9.2, took the largest contig and rotated it to a consistent starting position with [Dnaapler](https://github.com/gbouras13/dnaapler). I didn't polish with [Medaka](https://github.com/nanoporetech/medaka).[^nomedaka] I then counted the differences[^diff] between each assembly and the Illumina-polished ground truth I made previously.




## Results

<p align="center"><picture><source srcset="/assets/images/accuracy-vs-depth_update-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/accuracy-vs-depth_update.png" alt="Accuracy vs depth plot" width="100%"></picture></p>


This table compares the peak[^peak] Flye accuracy to the Trycycler accuracy (from my [last post](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html)):

| Genome                      | Flye                  | Trycycler          |
|:---------------------------:|:---------------------:|:------------------:|
| _Campylobacter jejuni_      | 42.65 errors<br>Q46.2 | 27 errors<br>Q48.2 |
| _Escherichia coli_          | 91.60 errors<br>Q47.5 | 70 errors<br>Q48.7 |
| _Listeria monocytogenes_    |  1.25 errors<br>Q63.7 |  0 errors<br>Q∞    |
| _Salmonella enterica_       | 71.75 errors<br>Q48.2 |  8 errors<br>Q57.8 |
| _Vibrio parahaemolyticus_   | 19.70 errors<br>Q52.2 |  7 errors<br>Q58.7 |




## Discussion and conclusions

First, it's important to keep in mind that these read sets were very clean and assemblable. This is because they already went through QC and Trycycler partitioning[^qc] – there were no short or junky reads, so every read was in principle valuable for the assembler. That means a 100× depth set in this analysis might be equivalent to a real-world 150× set, because after throwing out short and low-quality reads, you could lose 1/3 of your data.

That being said, I was still impressed with Flye. It often gave a decent assembly at very low depth (<20×). And it only produced a large-scale error in ~5% of its assemblies (the low points in the plot), i.e. ~95% of the assemblies were structurally perfect, containing only small-scale errors.

You can see in the plot that accuracy improved up to ~100× depth, after which additional reads brought no benefit. In fact, some of the genomes got a bit _worse_ with higher depth, which was surprising.[^highdepth] This suggests that if you have very-high-depth read sets, subsampling them (e.g. with [Filtlong](https://github.com/rrwick/Filtlong)) before Flye assembly might benefit not just computational time but also sequence accuracy. But I want to stress that these results are specific to Flye and its consensus algorithm. Different assemblers (e.g. [Canu](https://github.com/marbl/canu)) would likely produce different accuracy-vs-depth curves that may not have this dropping-off-at-higher-depths effect.

While >100× depth didn't help with assembly accuracy in this test, there are still benefits to deep ONT sequencing. Deep read sets allow for more aggressive read-length filtering[^filtering] and they [help with Trycycler assembly](https://github.com/rrwick/Trycycler/wiki/How-read-subsampling-works). Notably, for each genome, the peak accuracy in this analysis was worse than the Trycycler accuracy in my last post. So when accuracy _really_ matters, I still recommend sequencing to a depth of 200× or more and assembling with Trycycler.




## Footnotes

[^onepergenus]: This saved some computational time. Also, the accuracy values within each multi-genome genus were reasonably consistent, so I didn't think including all nine would add much. For _Vibrio_, I kept _V. parahaemolyticus_ – this is because _V. cholerae_ had a long inverted repeat in its chromosome that made it tougher to assemble. For _Campylobacter_ and _Listeria_, I kept _C. jejuni_ and _L. monocytogenes_ because they are the more commonly studied species. The five genomes I used here are the same ones from my [May post](https://rrwick.github.io/2023/05/05/ont-only-accuracy-with-r10.4.1.html).

[^largest]: For most genomes, the largest replicon was the chromosome, and for the _Vibrio_ genome, it was the larger of the two chromosomes.

[^maxdepth]: Only the _Campylobacter_ and _Listeria_ genomes had >400× depth, so for the other three, I took the depth as high as I could go. _Vibrio_ was the shallowest with only 208× depth to work with.

[^nomedaka]: I skipped Medaka both to save time and because it doesn't seem to help much for sup-read assemblies.

[^diff]: I used my [`compare_assemblies.py`](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki/Comparing-assemblies) script to get a difference count.

[^peak]: To calculate peak accuracy for each genome, I used the mean error count from the 20 best assemblies.

[^qc]: The reads were first QCed with Filtlong: `--min_length 10000` to remove short reads then `--keep_percent 90` to discard the worst 10% of each read set. Then Trycycler partitioning served as an additional QC step, because any read which originated from something else (e.g. a plasmid or cross-barcode contamination) was discarded.

[^highdepth]: Note that my read subsampling was random (`seqtk sample`), so lower depth sets did not have a higher average quality (as would have been the case if I had used Filtlong to subsample).

[^filtering]: For example, if you have a very deep read set, you can throw out all reads <10 kbp and still have plenty left over. This generally makes assembly easier, though small plasmids can be lost.

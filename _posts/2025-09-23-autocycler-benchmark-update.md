---
layout: post
title:  "Benchmark update: metaMDBG and Myloasm"
date:   2025-09-23
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.17180612.svg)](https://doi.org/10.5281/zenodo.17180612)


## New assembler releases

Our manuscript describing Autocycler was recently published:[^advance]<br>
[Wick RR, Howden BP, Stinear TP (2025). Autocycler: long-read consensus assembly for bacterial genomes. _Bioinformatics_. doi:10.1093/bioinformatics/btaf474.](https://doi.org/10.1093/bioinformatics/btaf474)

But its benchmarking is already out of date! Since I ran the analyses for the paper, two long-read assemblers have had new releases: [metaMDBG](https://github.com/GaetanBenoitDev/metaMDBG) v1.2 and [Myloasm](https://github.com/bluenote-1577/myloasm) v0.2.0. Both came with claims that caught my eye ('improved assembly quality' for metaMDBG and 'cleaner contig outputs with better polishing' for Myloasm), and both tools are still young (especially Myloasm). I therefore decided to rerun these new versions through the same benchmarking pipeline I used in the Autocycler paper.[^github]



## Updated results

Below is an updated version of Figure 2 from the Autocycler paper. Error counts are shown on the y-axes (pseudo-log transformed, lower is better). The original metaMDBG and Myloasm versions (from the paper) are orange, the new versions are green and everything else (less relevant here) is grey.

<p align="center"><picture><source srcset="/assets/images/autocycler_update_figure_2-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/autocycler_update_figure_2.png" alt="Autocycler benchmark updated results" width="100%"></picture></p>

I also updated the relevant supplementary figures using the same old-orange new-green colour scheme:
* <a href="/assets/images/autocycler_update_figure_s1.png" target="_blank">Figure S1: detailed results by error type</a>  
* <a href="/assets/images/autocycler_update_figure_s2.png" target="_blank">Figure S2: Inspector results</a>  
* <a href="/assets/images/autocycler_update_figure_s3.png" target="_blank">Figure S3: CRAQ results</a>  
* <a href="/assets/images/autocycler_update_figure_s4.png" target="_blank">Figure S4: BUSCO results</a>  
* <a href="/assets/images/autocycler_update_figure_s5.png" target="_blank">Figure S5: runtime and memory usage</a>  



## Discussion

Both metaMDBG and Myloasm showed clear improvements in accuracy with their latest releases: fewer sequence errors (substitutions and indels) and fewer total structural errors.[^structural] I was particularly impressed by the best cases for Myloasm v0.2.0 – a couple of the _Listeria innocua_ assemblies had only one single-bp error, better than any other single-tool assembler.

When I run Autocycler, I usually use [this Bash script](https://github.com/rrwick/Autocycler/tree/main/pipelines/Automated_Autocycler_Bash_script_by_Ryan_Wick) to automate the process. Autocycler benefits from a diverse set of input assemblers, but I had previously left out Myloasm because v0.1.0 had relatively high error rates. These new results, along with positive reports from a colleague[^michael], convinced me to add Myloasm to the pipeline.

It's worth noting that both metaMDBG and Myloasm were developed as _metagenome_ assemblers, but I'm using them here to assemble isolate genomes. As my results show, metagenome assemblers can work quite well on isolates! However, they can be more likely to leave low-depth contigs in the assembly. In metagenomes this is desirable, since there are often many low-abundance organisms. But for isolates, low-depth contigs usually indicate contamination.[^contamination] For these tests, I ran the assemblies via [Autocycler helper](https://github.com/rrwick/Autocycler/wiki/Autocycler-helper) using `--min_depth_rel 0.1`{:.nowrap} to remove contigs below 10% chromosomal depth, and I recommend others do the same when applying these assemblers to isolates.



## Footnotes

[^advance]: At the time of writing, the paper is reviewed and accepted but still an unproofed advance article. 

[^github]: For the full methods and results, see the [Autocycler paper GitHub repo](https://github.com/rrwick/Autocycler-paper/tree/main/2025-09_update).

[^structural]: The only metric that got worse with the new versions is 'missing bases', but this was balanced by improvements in the 'extra bases' metric (see <a href="/assets/images/autocycler_update_figure_s1.png" target="_blank">Figure S1</a>). 

[^michael]: [Michael Hall](https://github.com/mbhall88) had one tricky genome where metaMDBG and Myloasm were the only two assemblers which could successfully assemble the chromosome.

[^contamination]: A common cause would be cross-barcode contamination. In multiplexed ONT runs, some reads can 'leak' into other barcodes, and if the source is sufficiently high depth (e.g. a high-copy-number plasmid), the contamination can sometimes reach assemblable levels in wrong barcodes.

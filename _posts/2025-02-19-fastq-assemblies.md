---
layout: post
title:  "FASTQ assemblies with Dorado polish"
date:   2025-02-19
author: Ryan Wick
mathjax: true
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.14890690.svg)](https://doi.org/10.5281/zenodo.14890690)

This post builds on my [previous one](https://rrwick.github.io/2025/02/07/dorado-polish.html) about using [Dorado](https://github.com/nanoporetech/dorado) for genome polishing. Its key takeaway: Dorado polish and Medaka use the same bacterial polishing model, making them effectively interchangeable for bacterial genomes (at least at the time of writing).

While writing that post, I noticed something interesting in the Dorado docs: using the `--qualities` option outputs the polished assembly in FASTQ format. I initially thought this was unique to Dorado, but I checked Medaka, and it actually has a `-q` option that does the same thing, so this feature has been hiding under my nose for a while now.[^medakaq]

The key difference between FASTA and FASTQ is that FASTQ includes per-base quality scores, encoded as ASCII characters.[^ascii] In a well-calibrated FASTQ, these scores indicate the absolute probability of an error, but if not well-calibrated they can still indicate relative error likelihood.

FASTQ is commonly used for reads, while assemblies typically use FASTA. But assemblies _can_ use FASTQ format, and it could be useful in downstream analyses. For example, when calling variants from an assembly, errors can create false positives.[^arereadsrequired] FASTQ quality scores could help by allowing one to mask low-confidence bases in the assembly.

So what do Dorado polish FASTQs look like? How well do low qscores correlate with assembly errors? Are the scores well calibrated? In this post, I take an initial look using the same genomes from my [previous post](https://rrwick.github.io/2025/02/07/dorado-polish.html), focusing on the five that still had errors after Dorado-bac polishing.



## Calibration

To assess how well the FASTQ quality scores are calibrated, I calculated the expected number of errors per genome using
$$
\sum 10^{-q/10}
$$
summing over all genomic positions.

<div style="max-width: 500px; margin: auto;">
  <table>
    <thead>
      <tr><th style="text-align: left;">Genome</th><th style="text-align: right;">Actual<br>errors</th><th style="text-align: right;">Expected<br>errors</th></tr>
    </thead>
    <tbody>
      <tr><td><em>Shigella flexneri</em></td><td style="text-align: right;">5</td><td style="text-align: right;">58.4</td></tr>
      <tr><td><em>Klebsiella pneumoniae</em></td><td style="text-align: right;">3</td><td style="text-align: right;">19.4</td></tr>
      <tr><td><em>Providencia rettgeri</em></td><td style="text-align: right;">2</td><td style="text-align: right;">16.9</td></tr>
      <tr><td><em>Enterobacter kobei</em></td><td style="text-align: right;">11</td><td style="text-align: right;">31.7</td></tr>
      <tr><td><em>Escherichia coli</em></td><td style="text-align: right;">4</td><td style="text-align: right;">23.5</td></tr>
    </tbody>
  </table>
</div>

While not perfectly calibrated, the predictions are within an order of magnitude or so – better than I expected!



## Qscore plots at error locations

Across the five genomes, there were 25 bp of errors at 17 loci. Each locus is plotted below, with qscores on the y axes. The x axes display the assembled sequence (top) and ground truth (bottom, sometimes with extra bases squeezed in when the assembly had a deletion). Red bars mark error positions, and for deletion errors I used red on both sides of the deletion.

<p align="center"><picture><source srcset="/assets/images/dorado_polish_qscore_errors-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/dorado_polish_qscore_errors.png" alt="Per-base qscores at error loci" width="100%"></picture></p>

As you can see, positions with or near errors often have a much lower qscore than their neighbours. This is excellent, as it means the FASTQ scores indeed provide a useful indicator of base reliability.

However, there are exceptions. In particular, **E** and **F** lacked qscore drops. On closer inspection, the ONT reads at these sites were very clear (no heterogeneity) but differed from the Illumina reads, suggesting these might not be errors at all.[^nonerrors] Since my ONT and Illumina reads came from separate DNA extractions, biological differences between the read sets are possible. Even discounting **E** and **F**, some errors showed only subtle qscore drops. For example, **C** and **G** appear to be real errors, yet their qscores remain above 40.



## Discussion

These results just scratch the surface, but they show that Dorado-polish qscores, while imperfect, are potentially quite useful. Using my previous example of calling variants from an ONT assembly, one could reduce false positives by masking low-quality bases (perhaps with a local threshold, e.g. masking bases with a qscore significantly lower than their neighbours). The ideal strategy would balance sensitivity and specificity based on the application.

FASTQ assemblies could also improve genome polishing. I've spent a lot of time trying to make short-read polishing more reliable,[^howlow] but false-positive corrections can still occur.[^separate] Dorado-polish qscores could help decide which changes to accept. For example, **E** and **F** were short-read polishing corrections in the _Providencia_ genome, but their high Dorado-polish qscores could have convinced me to reject those changes.

Even if FASTQ output is a niche feature, I'm glad Dorado includes it. FASTQ assemblies provide a useful way to assess base-level accuracy, even when qscore calibration isn't perfect. I think we should embrace FASTQ as an assembly format, especially for workflows that could benefit from identifying unreliable bases.



## Footnotes

[^medakaq]: The `-q` option was added to `medaka_consensus` in v1.8.1 (June 2023), but it's undocumented in the Medaka README, which is probably why I missed it.

[^ascii]: This is usually done with this formula: `ascii - 33 = qscore`. So the scale starts with the `!` character (ascii = 33, qscore = 0) and could potentially go up to the `~` character (ascii = 126, qscore = 93), but the highest value I saw in a Dorado polish FASTQ was `g` (ascii = 103, qscore = 70).

[^arereadsrequired]: I'm about to release a preprint on this very topic – stay tuned!

[^nonerrors]: This means the error totals in my [previous post](https://rrwick.github.io/2025/02/07/dorado-polish.html) are slightly off. I reported the Dorado-bac assemblies as having 25 total errors, but I now think 23 total errors is more accurate.

[^howlow]: George Bouras and I published this paper last year with a lot of relevant info: [How low can you go? Short-read polishing of Oxford Nanopore bacterial genome assemblies.](https://doi.org/10.1099/mgen.0.001254)

[^separate]: This is especially true when there are genuine biological differences between the short-read and long-read sets, as is likely the case for the _Providencia_ genome in this post. I hate hybrid read sets made from separate DNA extractions!


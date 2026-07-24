---
layout: post
title:  "Benchmark update: Ilesta, Autocycler-fast and new versions"
date:   2026-07-24
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21525431.svg)](https://doi.org/10.5281/zenodo.21525431)

It's been 10 months since I [last posted](https://rrwick.github.io/2025/09/23/autocycler-benchmark-update.html) an update to my Autocycler benchmarks, and there have since been a few changes to the long-read assembler landscape:
- [Hybracter](https://github.com/gbouras13/hybracter) has been updated to v0.14.0.
- [metaMDBG](https://github.com/GaetanBenoitDev/metaMDBG) has been updated to v1.4.
- [Myloasm](https://github.com/bluenote-1577/myloasm) has been updated to v0.6.0.
- [Ilesta](https://github.com/yvlaere/Ilesta) is a new long-read assembler.
- I've made some tweaks to my [Autocycler bash pipeline](https://github.com/rrwick/Autocycler/tree/main/pipelines/Automated_Autocycler_Bash_script_by_Ryan_Wick), including a new fast version.

For this post, I re-ran the [Autocycler paper](https://doi.org/10.1093/bioinformatics/btaf474) benchmarks, covering all of the above.[^metrics]




## Updated results

All of the plots in this post use a pseudo-log transform on the y-axis.

Below is an updated Figure 2 from the Autocycler paper, with current versions for all tools.[^curated] Error counts are shown on the y-axes (lower is better):

<p align="center"><picture><source srcset="/assets/images/autocycler_update_2_figure_2-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/autocycler_update_2_figure_2.png" alt="Autocycler benchmark updated results" width="100%"></picture></p>

For a detailed breakdown of error types, see this updated Figure S1: <a href="/assets/images/autocycler_update_2_figure_s1.png" target="_blank">detailed results by error type</a>.

This is an updated Figure S5, showing runtime and memory usage (lower is better):
<p align="center"><picture><source srcset="/assets/images/autocycler_update_2_figure_s5-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/autocycler_update_2_figure_s5.png" alt="Autocycler benchmark updated results" width="100%"></picture></p>

And for the tools with updated versions, here are some plots that show their metrics across versions: <a href="/assets/images/autocycler_update_2_hybracter.png" target="_blank">Hybracter</a>, <a href="/assets/images/autocycler_update_2_metamdbg.png" target="_blank">metaMDBG</a> and <a href="/assets/images/autocycler_update_2_myloasm.png" target="_blank">Myloasm</a>.[^versions]




## Discussion

Of the tools with new versions, I was particularly impressed with metaMDBG. The current version (v1.4) produces fewer assembly errors than previous versions, and its low values for missing bases and extra bases make its total errors (bottom plot of Figure 2) the lowest of all the single-tool assemblers. It was already the most memory-efficient assembler I tested, and its runtime is now half of what it used to be. Very nice!

For both Hybracter and Myloasm, their current accuracy is about the same as when I last tested them. But Myloasm's memory usage and runtime have improved, so it's now tied with Raven as the fastest assembler tested.

Ilesta is the new kid on the block, and I've now added it to [Autocycler helper](https://github.com/rrwick/Autocycler/wiki/Autocycler-helper). Like miniasm, Ilesta produces an unpolished assembly, so like miniasm, I ran its output through [Minipolish](https://github.com/rrwick/Minipolish). In the benchmark, it didn't stand out in any particular metric, and its maximum total error count was the highest of the assemblers tested (it really struggled with the _Shigella_ genome). So I'm not going to include it in my Autocycler pipeline yet, but the tool is young, so I'll keep an eye on it and maybe reconsider if there are future versions.

Finally, this blog post introduces a new fast version of my [Autocycler bash pipeline](https://github.com/rrwick/Autocycler/tree/main/pipelines/Automated_Autocycler_Bash_script_by_Ryan_Wick). An Autocycler assembly can take a long time to run, not because of Autocycler itself, but because of the time it takes to make the input assemblies. The [`autocycler_full.sh`](https://github.com/rrwick/Autocycler/blob/main/pipelines/Automated_Autocycler_Bash_script_by_Ryan_Wick/autocycler_full.sh) pipeline generates 36 input assemblies (9 assemblers × 4 read subsets), and much of that time is spent on Canu (the slowest assembler). The new [`autocycler_fast.sh`](https://github.com/rrwick/Autocycler/blob/main/pipelines/Automated_Autocycler_Bash_script_by_Ryan_Wick/autocycler_fast.sh) pipeline generates just 12 input assemblies (6 assemblers × 2 read subsets). To save time, it does not run Canu (the slowest assembler), but it still runs [Plassembler](https://github.com/gbouras13/plassembler) (also pretty slow) to recover small plasmids that other assemblers cannot. If you don't care about small plasmids, you could drop Plassembler and save more time.

In this benchmark, the fast Autocycler pipeline was just as accurate as the full Autocycler pipeline, and it cut the runtime by more than three-quarters in most cases, making it comparable to Hybracter's runtime. I still think the full pipeline will sometimes be beneficial, since I do encounter genomes where including Canu helps. But the fast pipeline should be perfectly suitable for most genomes, so some users may want to switch.




## Footnotes

[^metrics]: But this time I didn't run the additional assessment tools from the Autocycler paper: Inspector (Figure S2), CRAQ (Figure S3) and BUSCO (Figure S4).

[^curated]: I didn't include Autocycler-curated results in this benchmark because they weren't needed! The plasmid which was occasionally missing in earlier benchmarks (and thus necessitated manual intervention) was not missing in these runs. So the automated Autocycler assemblies were all fine on their own.

[^versions]: I didn't test every version of these tools, just the current version plus any benchmark results from previous versions that I already had, either from the Autocycler paper or the last update post.

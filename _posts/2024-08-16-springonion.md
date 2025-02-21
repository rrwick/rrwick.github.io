---
layout: post
title:  "Spring OnION: a high-spec laptop for ONT sequencing"
date:   2024-08-16
modified_date: 2025-02-21
author: Ryan Wick, Louise Judd
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.13329198.svg)](https://doi.org/10.5281/zenodo.13329198)



Last year, I shared [this post](https://rrwick.github.io/2023/12/09/ont-desktop.html) about our ONT-sequencing desktop, OnION. It's been working well, but Louise and the [DAMG](https://www.doherty.edu.au/genomics) team often need to travel, so we've recently set up an ONT-sequencing laptop named Spring OnION. This post shares its details and performance to help others who might be considering a similar setup.

<p align="center"><picture><img src="/assets/images/springonion.jpg" alt="ONT sequencing laptop" width="50%"></picture></p>



## Computer details

Laptops don't offer as much opportunity for custom builds as desktops, so instead of buying Spring OnION from a boutique PC shop, we opted for a major manufacturer. We chose the Lenovo Legion Pro 7i 16", purchased for 5070 AUD (on sale at the time).

Here are Spring OnION's key specs:
* Intel i9-14900HX CPU (32 threads)
* 32 GB RAM
* NVIDIA RTX 4090 Laptop GPU
* 2TB SSD

While we often do bioinformatics on OnION, Spring OnION is primarily for sequencing, so we decided to leave Windows 11 as the OS rather than installing Linux. With [WSL](https://learn.microsoft.com/en-us/windows/wsl/about), command-line Linux is easy to use on Windows, so it still has plenty of bioinformatics capability. It's been a while since I regularly used a Windows machine, and I was pleasantly surprised to find that the native Windows command-line (via the [Windows Terminal](https://learn.microsoft.com/en-us/windows/terminal)) feels more Linux-like than it used to.[^terminal]



## Basecalling performance

To test Spring OnION's basecalling performance, I grabbed 10 pod5 files from a recent run. These contained 40k reads, ~206 Mbp of sequence and had an N50 length of ~10.5 kbp. I basecalled them with [Dorado](https://github.com/nanoporetech/dorado) v0.7.2 using the current hac and sup models on Spring OnION, OnION and the university's HPC. Here are the times (min:sec) for basecalling to complete:

|                                    | Spring OnION<br>(RTX 4090 Laptop) | OnION<br>(RTX 4090) | HPC<br>(A100) |
|:----------------------------------:|:---------------------------------:|:-------------------:|:-------------:|
| dna_r10.4.1_e8.2_400bps_hac@v5.0.0 | 2:20                              | 1:09                | 0:57          |
| dna_r10.4.1_e8.2_400bps_sup@v5.0.0 | 28:39                             | 11:18               | 6:24          |


As you can see, Spring OnION has about 40–50% of OnION's basecalling performance. We've successfully run four MinIONs simultaneously on OnION with live sup basecalling, so I am confident that Spring OnION can handle two MinIONs at once.

That being said, at the time of writing, MinKNOW is still using the sup@v4.3.0 basecalling model, not the current sup@v5.0.0 model. The latter has shifted to [using transformers](https://nanoporetech.com/blog/transforming-basecalling-in-genomic-sequencing), which has increased accuracy but runs considerably slower.[^v5] So if and when MinKNOW starts using v5.0.0 models, this may reduce our number of simultaneous real-time sup-basecalling MinIONs.

Finally, despite having similar names, the [RTX 4090](https://www.techpowerup.com/gpu-specs/geforce-rtx-4090.c3889) and the [RTX 4090 Laptop](https://www.techpowerup.com/gpu-specs/geforce-rtx-4090-mobile.c3949) are emphatically _not_ the same GPU. The desktop version is a larger chip and clocked faster, giving it more than twice the performance of the laptop version. This isn't surprising, as desktops have the room for more power and cooling, but the naming scheme is misleading. Don't be fooled!



## Basecalling performance – Feb 2025 update

[Dorado v0.9.1](https://github.com/nanoporetech/dorado/releases/tag/v0.9.1) was released in Jan 2025, and its release notes include this comment:
> This release of Dorado brings significant basecalling speed improvements for Nvidia GPUs with compute capabilities 8.6 (Ampere – e.g., RTX A6000), 8.7 (Ampere – e.g., Orin family), and 8.9 (Ada Lovelace).

At the suggestion of ONT's Mike Vella, I re-ran the above benchmarks using Dorado v0.9.2. Since the same models were used, the basecalled reads are mostly unchanged.

|                                    | Spring OnION<br>(RTX 4090 Laptop) | OnION<br>(RTX 4090) | HPC<br>(A100) |
|:----------------------------------:|:---------------------------------:|:-------------------:|:-------------:|
| dna_r10.4.1_e8.2_400bps_hac@v5.0.0 | 1:11                              | 0:32                | 0:31          |
| dna_r10.4.1_e8.2_400bps_sup@v5.0.0 | 11:50                             | 4:11                | 4:13          |

Compared to Dorado v0.7.2, the HPC (A100) was 50% faster for the sup model – a welcome improvement.[^A100] More impressively, both OnION and Spring OnION (which use NVIDIA compute capability 8.9 GPUs) now deliver more than twice their previous sup-basecalling speed!

When first introduced in Dorado v0.7.0, the sup@v5.0.0 model was noticeably slower than its sup@v4.3.0 predecessor, but Dorado v0.9.1 compensates for that. These improvements are currently limited to command-line basecalling, since the update hasn't yet been integrated into the Dorado basecall server used by MinKNOW. But this new enhanced speed will eventually make its way to MinKNOW, which will help keep real-time sup-basecalling for simultaneous MinION runs feasible on both OnION and Spring OnION.



## Footnotes

[^terminal]: The commands I used for Dorado were the same for Linux and Windows. I also found that basic command-line navigation in Windows is more Linux-like than it used to be. For example, I can use `ls` instead of `dir` to view the contents of the current directory. It's not quite the same as the Linux `ls` command, but close enough, and it's nice to be able to use my muscle memory.

[^v5]: On the [release notes page](https://community.nanoporetech.com/posts/dorado-0-7-0-release) (login required), ONT said, 'In this initial release, the v5 SUP models are expected to run more slowly than v4.3 models. We will add speed enhancements over the coming months.'

[^A100]: This boost may have arrived with Dorado v0.8.0 (Sep 2024), as this version's release notes say 'improves the speed of v5 SUP basecalling models on A100/H100 GPUs'. Also, the A100 has compute capability 8.0, so it may not be affected by the Dorado v0.9.1 changes, but I didn't test this.

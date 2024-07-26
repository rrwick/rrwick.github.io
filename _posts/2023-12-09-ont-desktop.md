---
layout: post
title:  "OnION: a high-spec desktop for ONT sequencing"
date:   2023-12-09
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10311238.svg)](https://doi.org/10.5281/zenodo.10311238)



I work with [Doherty Applied Microbial Genomics](https://www.doherty.edu.au/genomics), a laboratory at the University of Melbourne which does microbial genomics research predominantly through collaborative partnerships. We do a lot of ONT sequencing, which up until recently has been on a GridION. However, GridION warranty costs are steep ([12500 USD per year](https://store.nanoporetech.com/device-warranty.html)), so we decided to replace it with a powerful desktop computer.

In this post, I share details about our new computer along with some setup tips and performance stats, in the hope that it might be useful to others going down the same road.

All ONT sequencers end in 'ION', but since our computer isn't from ONT, it's sort of an un-ION. So we named it 'OnION'. Credit to Taylor Harshegyi for coming up with that one :smile:

Here's a photo of OnION[^background] set up in the lab:
<p align="center"><picture><img src="/assets/images/onion.jpg" alt="ONT sequencing desktop" width="70%"></picture></p>




## Computer details

We bought OnION from [Aftershock PC Australia](https://www.aftershockpc.com.au), a Melbourne company that mainly specialises in flashy gaming computers full of RGB, but they also sell less-flashy workstations with less (but still some) RGB.

Here are OnION's key specs, all of which surpass the GridION it replaced:
* Intel i9-13900K CPU (32 threads)
* 128 GB of RAM
* NVIDIA RTX 4090 GPU
* 10 TB of SSD storage (2 TB Samsung 990 Pro and 8 TB PNY CS3140)

We also got [this USB hub](https://www.orico.cc/us/product/detail/7407.html) for connecting the MinIONs. It has high bandwidth and external power, so can hopefully handle four at once.[^hub]

The total cost was less than 8000 AUD (5300 USD). Gaming hardware is so much cheaper than workstation hardware! A 'proper' workstation with similar performance would have been three times the price.[^workstation]




## Setup

I tried to mimic the software setup of the GridION as much as possible. The OS is Ubuntu 20.04. For drive partitions, I gave about 7 TB of our big SSD to `/data` and the entirety of our 2 TB SSD to `/data/scratch`.

For some mysterious reason, the Ubuntu GUI didn't work at first, and I had to follow [these instructions](https://askubuntu.com/a/320696) to fix it.

I followed [these instructions](https://developer.nvidia.com/cuda-12-0-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=20.04&target_type=deb_local) to install CUDA 12 and [these instructions](https://community.nanoporetech.com/protocols/experiment-companion-minknow/v/v/installing-minknow-on-linu) to install MinKNOW. I then set a few paths in the `/opt/ont/minknow/conf/user_conf` file to make use of our drive partitions:
* `base` → `/data`
* `intermediate` → `scratch/intermediate`
* `complete_intermediate_reads` → `scratch/queued_reads`
* `reads_tmp` → `scratch/reads_tmp`

We ran into a 'flow cell not detected' problem in MinKNOW which was solved by making the MinKNOW service run as root.[^root] Since then, everything has been working smoothly.

I also installed a lot of common bioinformatics tools (e.g. [minimap2](https://github.com/lh3/minimap2), [Flye](https://github.com/fenderglass/Flye), etc). With all of OnION's CPU and RAM, we plan on using it for some on-board analysis.




## Basecalling performance

To see how fast OnION could basecall, I did a small benchmarking test. I took a single pod5 file (4000 reads, 28 Mbp) and ran it through [Dorado](https://github.com/nanoporetech/dorado) v0.4.2 on a variety of computers with different GPUs. I tried all three levels (fast, hac and sup) with five trials of each, taking the best result.

This plot shows the basecalling speed (higher is better), as reported by Dorado in its stderr:

<p align="center"><picture><source srcset="/assets/images/dorado_speed-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/dorado_speed.png" alt="Dorado basecalling speed" width="95%"></picture></p>


This plot shows the basecalling wall time (lower is better), as reported by [`time`](https://www.gnu.org/software/time):

<p align="center"><picture><source srcset="/assets/images/dorado_time-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/images/dorado_time.png" alt="Dorado basecalling time" width="95%"></picture></p>

I was very happy with OnION's performance! Extrapolating the results to a full MinION run (about 500× more data than my little test), it can complete sup basecalling in a few hours, so it's more than capable of real-time sup basecalling four simultaneous MinIONs. Also notable is that OnION basecalls at more than twice the speed of the GridION.

Basecalling speed and time were mostly correlated but not perfectly so. For example, on hac and sup basecalling, OnION had the best wall time but the HPC A100 node had higher Dorado-reported basecalling speeds. Perhaps IO performance is behind that discrepancy, and a larger test (more pod5s) might give more consistent results.[^a100]

I was also pleasantly surprised with the performance of my MacBook (M2 Pro, 16 GPU cores, 16 GB unified memory). While it can't compete with discrete NVIDIA GPUs and sup basecalling was very slow, it could manage real-time hac basecalling for one MinION.




## Footnotes

[^background]: The background image was made by DALL·E. I asked for 'a futuristic onion on a dark background'.

[^hub]: At the time of writing, we've run two MinIONs simultaneously through the USB hub with no problems, but we haven't yet tested all four at once.

[^workstation]: For comparison, I configured a computer with a Xeon W5-3435X CPU, 128 GB of ECC RAM and an RTX 6000 Ada GPU on Aftershock's site. I think this would perform about the same as OnION, and it cost almost 24000 AUD.

[^root]: [This post](https://community.nanoporetech.com/posts/update-and-installation-is) on the ONT community site details how to fix this. Searching for 'minknow.service' on the community site yields other posts with similar problems/solutions.

[^a100]: After I initially posted this, [Mike Vella clarified](https://twitter.com/vellamike/status/1733296996914446673) that an A100 should indeed be faster than an RTX 4090, and the discrepancy I saw (better wall time with OnION) is indeed due to IO performance and the small scale (just one pod5 file) of my test.

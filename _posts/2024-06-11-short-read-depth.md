---
layout: post
title:  "Short-read depth recommendations for polishing"
date:   2024-06-11
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.11556568.svg)](https://doi.org/10.5281/zenodo.11556568)


ONT-only bacterial genome assemblies now regularly have <10 errors (see my [last post](/2024/05/08/duplex_assemblies.html)), which makes short-read polishing less crucial than it used to be. [George Bouras](https://twitter.com/GB13Faithless) and [Matthew Croxen](https://twitter.com/m_croxen) were chatting on the µbioinfo Slack, and the question came up about how much short-read depth is necessary. If there are only a few errors to fix, can you use shallow short-read sequencing[^short] to save money?

I started experimenting with this in January 2024 with the intention of putting the results on this blog, and I used my current preferred polishing method: [Polypolish](https://github.com/rrwick/Polypolish) followed by [Pypolca](https://github.com/gbouras13/pypolca). Pypolca is a Python-based reimplementation of the [POLCA](https://github.com/alekseyzimin/masurca#polca) polisher made by George that's easier to install and run. However, I soon noticed that Pypolca could introduce a lot of errors at low depths. I also noticed some cases where Polypolish introduced errors at low depths, and this really bothered me, because I explicitly designed Polypolish to _not_ introduce errors.

Why am I so hung up on introduced errors? A few years ago, a good ONT-only bacterial genome assembly would contain hundreds to thousands of errors. If a polisher could fix those but introduced a few new errors in the process, that wasn't a big deal – it would still make the assembly much better. But with today's much-more-accurate ONT-only assemblies, a polisher that introduces errors can easily make an assembly _worse_, not better. So introduced errors now are a big deal!

It became clear that both Polypolish and Pypolca would need enhancements to avoid introducing errors when short-read depth is low. At this point, I decided the topic probably deserved a whole manuscript, not just a blog post. Since Pypolca is central to this, I teamed up with George. The paper introduces Pypolca and describes improvements to both Pypolca and Polypolish that help in low-depth scenarios, and we benchmarked these tools against [FMLRC2](https://github.com/HudsonAlpha/fmlrc2), [HyPo](https://github.com/kensung-lab/hypo), [NextPolish](https://github.com/Nextomics/NextPolish) and [Pilon](https://github.com/broadinstitute/pilon)[^polishers]. George and I split the analytical work (with help from the other authors, of course) and George did most of the writing. The manuscript is now published in Microbial Genomics:
> [**How low can you go? Short-read polishing of Oxford Nanopore bacterial genome assemblies**](https://doi.org/10.1099/mgen.0.001254)

As is often the case, this paper grew into something larger than originally planned, and it answers some other interesting questions as well. So please check it out for the full story! But in this post, I'd like to come back to the original question: when polishing a modern ONT-only assembly, is shallow short-read sequencing good enough?

The answer is... not really. 5× short-read depth can probably fix about ⅓ of the errors, and 10× depth can probably fix about ⅔ of the errors, but you need 25× or more to fix most of the errors. So I sadly cannot recommend shallow short-read sequencing for polishing. These results have, however, made me refine my recommended short-read depth. [In the past](https://doi.org/10.1371/journal.pcbi.1010905), I've suggested 100–300× short-read depth when aiming for a perfect assembly. Assuming all goes well on the ONT side, I now feel comfortable reducing that recommendation to 50× short-read depth.



## Footnotes

[^short]: I was in the habit of saying 'Illumina sequencing' because that used to be the only short-read game in town. But now that other platforms (e.g. [MGI](https://en.mgi-tech.com/products)) are becoming more common, I'm trying to retrain myself to say 'short-read sequencing' instead.

[^polishers]: I didn't include [ntEdit](https://github.com/bcgsc/ntEdit), [Racon](https://github.com/lbcb-sci/racon) and [wtpoa](https://github.com/ruanjue/wtdbg2) because those polishers performed worse than the rest when I benchmarked them in the [Polypolish paper](https://doi.org/10.1371/journal.pcbi.1009802).

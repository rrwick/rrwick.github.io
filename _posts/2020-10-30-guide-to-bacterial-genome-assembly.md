---
layout:        post
title:         "Choose-your-own-adventure guide to bacterial genome assembly"
date:          2020-10-30
modified_date: 2025-02-25
author:        Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7471199.svg)](https://doi.org/10.5281/zenodo.7471199)




<img align="right" width="245" style="padding:10px" src="/assets/images/choose_your_own_adventure.jpg" alt="Choose Your Own Adventure cover">

Welcome to this choose-your-own-adventure guide for assembling a bacterial genome! It will lead you to the best (in my opinion) strategy for getting a nice whole-genome assembly, based on what kind of reads you have and what your priorities are. And unlike in the [classic book series](https://en.wikipedia.org/wiki/Choose_Your_Own_Adventure), none of your choices will result in your untimely death!

This guide is for __bacterial isolate genomes__. I have little experience with eukaryote assembly, and so my recommendations may not hold for them. Eukaryote genomes are larger, more repetitive and usually diploid/polyploid, all of which place different demands on assemblers. This guide also isn't for metagenomes, which have their own set of assembly challenges (broad ranges of read depth and closely related genomes).

I have written this guide with the assumption that you already have reads for your to-be-assembled genome(s). If that's not true, check out this appendix: [what sequencing should I do?](#appendix-what-sequencing-should-i-do)

Ready to begin? [Start at step 1](#1-input-reads).
<br><br><br>




## History

This guide was originally posted on the [Trycycler wiki](https://github.com/rrwick/Trycycler/wiki) in October 2020, and there have been a few changes since then:
* November 2020: added comments about PacBio HiFi reads.
* February 2021: removed Shasta as a long-read assembler (too many errors).
* October 2021: changed short-read polishing instructions from Pilon to Polypolish/POLCA.
* June 2024: updated Unicycler commands and replaced POLCA with Pypolca.
* February 2025: replaced Trycycler with Autocycler and updated Medaka instructions.

<br>




## 1: Input reads

What type of reads do you have?
* Short reads (Illumina) only. [Go to 2](#2-short-read-qc).
* Long reads (Oxford Nanopore or PacBio) only. [Go to 7](#7-long-read-qc).
* Both short and long reads (hybrid). [Go to 15](#15-long-read-and-short-read-qc).

I don't have any experience with [BGI reads](https://www.bgi.com/global/resources/sequencing-platforms), but I'm told they are qualitatively similar to Illumina reads, so treat them as such in this guide. If you have BGI read assembly experience, please shoot me an email and share your knowledge!

<br>




## 2: Short-read QC

Before starting the actual assembly, I recommend a bit of read QC. My favourite tool at the moment is [fastp](https://github.com/OpenGene/fastp). It's easy to use and lives up to its name (very fast). I usually run it like this:
```bash
fastp --in1 input_1.fastq.gz --in2 input_2.fastq.gz --out1 short_1.fastq.gz --out2 short_2.fastq.gz --unpaired1 short_u.fastq.gz --unpaired2 short_u.fastq.gz
```

It will generate three files: `short_1.fastq.gz` and `short_2.fastq.gz` will contain paired reads and `short_u.fastq.gz` will contain reads which were orphaned by the QC process. Hopefully the `u` file is very small relative to the `1` and `2` files. If not, something might have gone wrong!

[Go to 3.](#3-short-read-assembly-priorities)

<br>




## 3: Short-read assembly priorities

Since you only have short reads, a complete genome assembly (one contig per replicon) is almost certainly not possible. Your assembly will be fragmented (a.k.a. a draft assembly) because short reads are insufficient to resolve repetitive regions of the genome :cry:

Now which of the following best applies to you?
* You want very accurate contigs with minimal misassemblies. You're willing to accept shorter contigs (i.e. a more fragmented assembly) if it reduces the risk of misassemblies. [Go to 4.](#4-skesa-assembly)
* You need a very reproducible assembly. E.g. the same read set should yield exactly the same assembly regardless of the number of threads or available memory. [Go to 4.](#4-skesa-assembly)
* You will be doing a _k_-mer-based analysis and nearly all of the genome's _k_-mers must be represented in the assembly. [Go to 5.](#5-spades-assembly)
* None of the above. You just want a nice bacterial genome assembly for general purposes. [Go to 6.](#6-unicycler-short-read-assembly)

<br>




## 4: SKESA assembly

For a very accurate and reproducible assembly, I would recommend [SKESA](https://github.com/ncbi/SKESA). Read more about it in [its paper](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-018-1540-z). It doesn't give the longest contigs, but its misassembly rate is low. You can run it like this:
```bash
skesa --reads short_1.fastq.gz,short_2.fastq.gz --cores 16 --memory 64 > skesa_assembly.fasta
```

I don't think SKESA can simultaneously take paired and unpaired reads, so we didn't use the `short_u.fastq.gz` here. But that file should only contain a tiny fraction of your total reads, so no big loss.

One thing to keep in mind is that SKESA can leave out a bit of sequence around tough-to-assemble parts of the genome. This is because it would rather give you no sequence than incorrect sequence. Your assembly can therefore have some gaps and some of the genome's _k_-mers may not be present in the assembly.

SKESA assemblies shouldn't require any polishing, so you're now finished! __THE END__

<br>




## 5: SPAdes assembly

If _k_-mer representation is important, I recommend [SPAdes](https://cab.spbu.ru/software/spades/), because its contigs usually have a bit of overlap with each other. E.g. two adjacent contigs at a point of fragmentation will share some sequence with each other. This means that _k_-mers at the breakpoint are still in the assembly.

You can run it like this:
```bash
spades.py -1 short_1.fastq.gz -2 short_2.fastq.gz -s short_u.fastq.gz -o spades_assembly --isolate --threads 16
```

Some things to keep in mind:
* SPAdes produces both a `contigs.fasta` file and a `scaffolds.fasta` file. The difference is that the latter may contain N bases, while the former will only have A, C, G and T. For many assemblies (especially if you have good read coverage), those two files may be identical.
* SPAdes produces an assembly graph (`assembly_graph.fastg`) but this is before any scaffolding takes place, so its sequences will probably be a bit shorter than those in `contigs.fasta`. While the `assembly_graph_with_scaffolds.gfa` file sounds like it's been scaffolded, it only contains descriptions of the paths that the scaffolds take in the graph – the sequences have not been merged into scaffolds.
* SPAdes does not filter out low-depth contigs by default, so low-level contamination can creep into your assembly. You may want to experiment with its `--cov-cutoff` option if that's a problem for you.
* SPAdes disables polishing when you use the `--isolate` option. If you want to do polishing, you can use the `--careful` option instead.
* I don't have much experience with [Shovill](https://github.com/tseemann/shovill), but that's a SPAdes-based assembly pipeline which takes care of a lot of things for you: subsampling reads to a lower depth, filtering out low-depth contigs, renaming contigs, etc. If you want to do a SPAdes assembly but not worry about the details, give it a try!

You're now finished! __THE END__

<br>




## 6: Unicycler short-read assembly

For most short-read bacterial genome assemblies, I'd recommend [Unicycler](https://github.com/rrwick/Unicycler). (Disclaimer: it's my own tool so I may be biased :smile:)

For short-read-only assemblies, Unicycler just runs SPAdes and then adds few extra touches:
* Filters out low-depth contigs by default to remove low-level contamination.
* Applies SPAdes scaffolding to the graph so you can get your final assembly in graph form.
* Normalises contig depth to the chromosomal level.
* Trims overlaps from the graph so contigs don't overlap (could be a bad thing if you're doing _k_-mer-based analysis, [see step 5](#5-spades-assembly)).

I usually run it like this:
```bash
unicycler -1 short_1.fastq.gz -2 short_2.fastq.gz -s short_u.fastq.gz -o unicycler_assembly --threads 16
```

Unicycler will produce its final assembly in FASTA format (`assembly.fasta`) and as a graph (`assembly.gfa`). They contain identical sequences, except the `assembly.fasta` file has very short (<100 bp by default) contigs excluded.

You're now finished! __THE END__

<br>




## 7: Long-read QC

I'm assuming here that you already have basecalled and demultiplexed long reads in FASTQ format. If not, you can check out the [Oxford Nanopore basecalling](#appendix-oxford-nanopore-basecalling) appendix.

Before continuing with long-read assembly, I often like to do some light QC on the reads with [Filtlong](https://github.com/rrwick/Filtlong):

```bash
filtlong --min_length 1000 --keep_percent 95 input.fastq.gz | gzip > long.fastq.gz
```

This will remove any reads shorter than 1 kbp and also exclude the worst 5% of reads. The goal is just to get the really bad reads out of your set.

I usually keep QC light for two reasons, the first being small plasmids. Filtlong considers shorter reads 'bad' and longer reads 'good' so more aggressive filtering will leave you with few reads on the short end of the spectrum. For most of the genome this is probably a good thing, but it can be disastrous for small plasmids. For example, if you have a big read set that you've aggressively filtered with Filtlong, you might be left with no reads smaller than 10 kbp. If that genome has a small plasmid 4 kbp in size, it will now be gone from the read set!

The second reason I keep QC light is that deep long-read sets can benefit assembly. This is especially true if you end up using Autocycler, where deeper sets allow for more independent input assemblies (see [this page on the Autocycler wiki](https://github.com/rrwick/Autocycler/wiki/Autocycler-subsample)). Other assemblers handle deep read sets just fine too, so when in doubt, aim for deep.

That being said, if you have an _extremely_ deep long-read set (e.g. 500×) then more aggressive QC is warranted. Just be careful about not losing all your shorter reads if small plasmids are important to you. Filtlong has some options which might help, like `--mean_q_weight` which can make it prioritise quality over length and `--min_mean_q` which can set a hard threshold for read quality.

[Go to 8.](#8-long-read-assembly-priorities)

<br>




## 8: Long-read assembly priorities

With long reads, you will probably be able to get a complete assembly: one contig per replicon in the genome. This relies on you having reads which are longer than the longest repeat in the genome, but for most bacterial genomes, this isn't hard. E.g. if your genome's longest repeat is 6 kbp and your reads have an N50 length of 10 kbp, assembly should proceed well. But keep in mind that some bacterial genomes do have very long repeats (e.g. 100 kbp) and complete assembly in such cases will require ultra-long reads.

Another thing to keep in mind: long reads have higher error rates than short reads, and since these errors aren't completely random, some can persist into the final assembly. So don't expect a 100% accurate assembly with long reads alone, though in many cases you can get pretty close. This applies to both Oxford Nanopore and PacBio reads, though PacBio can probably provide higher assembly accuracy.

With that out of the way, which of the following best applies to you?
* Computational performance is important. You have a lot of assemblies to do and want them done fast! [Go to 9.](#9-raven-assembly)
* Assembly accuracy is important. You're willing to accept a slower assembly if it's more accurate. [Go to 10.](#10-flye-assembly)
* You want the most accurate possible assembly and are willing to spend time and effort to get it. [Go to 11.](#11-autocycler-assembly)

<br>




## 9: Raven assembly

[Raven](https://github.com/lbcb-sci/raven) is not as fast as Shasta, but it's still pretty fast, and it usually produces better assemblies. [In my tests](https://f1000research.com/articles/8-2138) it was also very reliable and robust. You can run it like this:

```bash
raven --threads 16 long.fastq.gz > raven_assembly.fasta
```

Since you chose computational performance over accuracy, I'm going to assume that you're not interested in post-assembly polishing steps. So you're now finished! __THE END__

<br>




## 10: Flye assembly

While there are many different metrics with which one can evaluate [long-read assemblers](https://f1000research.com/articles/8-2138), [Flye](https://github.com/fenderglass/Flye) is an overall strong performer. Its main downside is that you'll need a bit more computational resources than you would for other assemblers. 32 GB of RAM and 1 hour should be sufficient for most read sets.

You can run Flye like this (or with `--pacbio-raw` as appropriate):
```bash
flye -o flye_assembly --plasmids --threads 16 --nano-raw long.fastq.gz
```

Flye's `--plasmids` option enables a nice feature which tries to recover small plasmids in the genome. However, it has a nasty habit of sometimes doubling small plasmids in a single contig. E.g. if your genome has a 4 kbp plasmid, Flye might create an 8 kbp contig with two whole copies of the plasmid sequence. Something to keep an eye out for!

When your Flye assembly is done:
* If sequence accuracy is not particularly important to your analyses, then a Flye assembly is probably enough and you're finished! __THE END__
* But polishing can significantly improve accuracy, so if that sounds appealing to you, [go to 12](#12-long-read-polishing).

<br>




## 11: Autocycler assembly

Running [Autocycler](https://github.com/rrwick/Autocycler) is more involved than other approaches, but it can yield the best assemblies. (Disclaimer: it's my own tool so I may be biased :smile:) I won't say more about it here – read more about it on [the Autocycler wiki](https://github.com/rrwick/Autocycler/wiki).

When your Autocycler assembly is done:
* If sequence accuracy is not particularly important to your analyses, then an Autocycler assembly is probably enough and you're finished! __THE END__
* But polishing can often improve accuracy, so if that sounds appealing to you, [go to 12](#12-long-read-polishing).

<br>




## 12: Long-read polishing

Long-read polishing can significantly improve sequence accuracy, and since long reads can span most repeats, long-read polishing can make repeats just as accurate as non-repetitive sequences. If you have a hybrid read set (both short and long reads), I recommend you do long-read polishing first and short-read polishing second.

[Racon](https://github.com/isovic/racon) is a platform-agnostic tool which can work with both Oxford Nanopore and PacBio reads. However, I've found that it's not usually necessary – most long-read assemblers produce sufficiently accurate sequence all by themselves. And some ([Raven](https://github.com/lbcb-sci/raven) and [Miniasm/Minipolish](https://github.com/rrwick/Minipolish)) have Racon integrated into their pipeline.

I therefore think it's appropriate to jump right to the platform-specific polishers:
* If you have Oxford Nanopore reads, [go to 13.](#13-medaka)
* If you have PacBio reads, [go to 14.](#14-pacbio-polishing)

<br>




## 13: Medaka

[Medaka](https://github.com/nanoporetech/medaka) is Oxford Nanopore's polishing tool, and it seems to work very well for cleaning up an ONT-only assembly. Conveniently, it operates on FASTQ reads and so does not require raw data. It's also pretty fast!

You can run it like this:
```bash
medaka_consensus -i long.fastq.gz -d input_assembly.fasta -o medaka --bacteria
```

Some things to keep in mind:
* Medaka has multiple trained models that it can use for polishing, but if your reads have basecalling information in the FASTQ file, then Medaka can automatically choose the best model.
* The `--bacteria` option makes Medaka prefer its bacterial-specific model (assuming compatibility with your basecalling), and this is usually the best choice.
* Structural errors in your assembly (such as missing plasmids or start-end overlap of circular contigs) can sometimes lead to Medaka _increaing_ the number of errors. It's therefore ideal to make sure that your input assembly is structurally correct before running Medaka.

After Medaka is done:
* If you only have long reads for your genome, then you're finished! __THE END__
* If you also have short reads (i.e. you're doing a hybrid assembly), then [go to 20](#20-short-read-polishing).

<br>




## 14: PacBio polishing

If you have older CLR PacBio reads (single-pass reads with lower accuracy), then you should use Arrow to polish your assembly. Arrow is PacBio's polishing algorithm, and you can run it via the [GCpp tool](https://github.com/PacificBiosciences/gcpp). I must confess that I have less experience with PacBio than Oxford Nanopore, but when I last used Arrow it gave very good sequence accuracy. However, it does require the raw read data (not FASTQ reads) which makes it more cumbersome to run.

If you have newer [HiFi PacBio reads](https://www.pacb.com/smrt-science/smrt-sequencing/hifi-reads-for-highly-accurate-long-read-sequencing), then polishing at this step is probably not necessary. That's because your PacBio platform has already run an Arrow-like algorithm on the individual reads (which are themselves consensus sequences). So consider your assembly already polished!

* If you only have long reads for your genome, then you're finished! __THE END__
* If you also have short reads (i.e. you're doing a hybrid assembly), then [go to 20](#20-short-read-polishing).

<br>




## 15: Long-read and short-read QC

Congratulations on having a hybrid read set – it bodes well for getting a genome assembly that is both accurate and complete! Before continuing, you should do some light QC on your reads. I won't repeat myself, so look back to [step 2 for short-read QC](#2-short-read-qc) and [step 7 for long-read QC](#7-long-read-qc).

One other thing to keep in mind: your short and long reads should be consistent with each other – i.e. they should be sampling from the exact same genome. See the [hybrid read set mismatch appendix](#appendix-hybrid-read-set-mismatch) for more about this.

[Go to 16.](#16-read-set-depths)

<br>




## 16: Read set depths

The best hybrid assembly approach will depend on the depths of your read sets. To calculate these values, you'll need to know the approximate genome size: depth = (bases in reads) / (bases in genome). For example, if your genome is 4 Mbp and you have 600 Mbp of reads, then your read depth is 150×.

Here are some arbitrary depth ranges we can use:
* shallow: 0–50×
* medium: 50–100×
* deep: >100×

Which of the following best applies to you?
* Your short-read set is deep but your long-read set is shallow. [Go to 17.](#17-unicycler-hybrid-assembly)
* Your long-read set is deep. [Go to 18.](#18-long-read-first-hybrid-assembly)
* Neither your short-read nor long-read sets are deep. [Go to 19.](#19-shallow-reads)

<br>




## 17: Unicycler hybrid assembly

[Unicycler](https://github.com/rrwick/Unicycler) is getting a bit long in the tooth, but its hybrid assembly approach is still appropriate for cases where long reads are limited ([read more on Trycycler's FAQ](https://github.com/rrwick/Trycycler/wiki/FAQ-and-miscellaneous-tips#should-i-use-unicycler-or-trycycler-to-assemble-my-bacterial-genome)).

Unicycler works by first generating a short-read assembly graph and then using long reads to scaffold the graph to completion. It performs best when the short reads are deep and have even coverage, as this makes for a clean graph on which it can work. Not many long reads are then required – as little as 10× can sometimes be enough.

Run a Unicycler hybrid assembly like this:
```bash
unicycler -1 short_1.fastq.gz -2 short_2.fastq.gz -s short_u.fastq.gz -l long.fastq.gz -o unicycler_assembly --threads 16
```

After making a short-read assembly graph, Unicycler tries to assign a multiplicity value to each contig: a multiplicity of 1 means single-copy (the contig occurs once in the genome) while a multiplicity of 2 or more means repeat (the contig occurs more than once in the genome). If Unicycler makes a mistake in this step, it can cause problems with the scaffolding which lead to an incomplete assembly. This is one of the most common reasons that Unicycler fails to completely assemble a nice hybrid read set. You therefore might want to check the double-check the multiplicity and fix any errors you see. Read more about this on [Unicycler's wiki](https://github.com/rrwick/Unicycler/wiki/Multiplicity-mistake).

You're now finished! __THE END__

<br>




## 18: Long-read-first hybrid assembly

Deep long reads allow for a long-read-first approach to hybrid assembly, which is my preference. You should be able to get a very nice genome sequence! You'll be doing a long-read-only assembly followed by short-read polishing.

[Go to 8.](#8-long-read-assembly-priorities)

<br>




## 19: Shallow reads

If both your short-read and long-read sets are shallow, then you're in a tough spot! Getting a nice assembly might not be easy. I would recommend trying both Unicycler's short-read-first approach ([step 17](#17-unicycler-assembly)) and a long-read-first approach ([step 19](19-long-read-first-hybrid-assembly)). Then compare the two resulting assemblies to see which looks better. Good luck!

<br>




## 20: Short-read polishing

Even after long-read polishing, there are still probably some errors in your bacterial genome, mainly homopolymer-length errors. Since Illumina reads don't suffer from the same types of errors, you can use short-read polishing to clean these up and get a very accurate assembly.

[Polypolish](https://github.com/rrwick/Polypolish) is a tool I've written for this purpose, and I also quite like [Pypolca](https://github.com/gbouras13/pypolca). Read more about these tools here: [How low can you go? Short-read polishing of Oxford Nanopore bacterial genome assemblies.](https://doi.org/10.1099/mgen.0.001254)

Can short-read polishing make your genome perfect, i.e. with absolutely no errors? If your read sets were good and there were no complications along the way, then this is often possible. The most challenging errors to fix are those in repeats, and they can be minimised by getting your assembly as good as possible before short-read polishing. This is why long-read polishing is important (e.g. Autocycler+Medaka+Polypolish+Pypolca should do better than Autocycler+Polypolish+Pypolca).

After short-read polishing:
* If you don't care about small plasmids (or if you happen to know that your genome doesn't contain small plasmids), then you're finished! __THE END__
* If small plasmids are important to you, [go to 21.](#21-small-plasmid-recovery)

<br>




## 21: Small-plasmid recovery

Small plasmids can sometimes be underrepresented in long-read sets, especially if your read lengths are long and you used a ligation-based sample prep (read more about this in the [appendix below](#long-read-sequencing)). This means that your assembled genome could be missing one or more small plasmids if you've done a long-read-first hybrid assembly.

To solve this problem, I'd recommend you do a Unicycler hybrid assembly (see [step 17](#17-unicycler-hybrid-assembly)) of your genome. See if any small plasmids appear in the Unicycler assembly that aren't in your long-read-first assembly. If so, manually copy them over and then you're finished! __THE END__

<br>





<br>

## Appendix: what sequencing should I do?

If you're reading this page before you've done any sequencing, then I applaud your foresight! The best sequencing approach will depend on your downstream analyses, and limited funds can lead to tough trade-offs (e.g. deep sequencing of fewer isolates or shallow sequencing of more isolates). This is a big topic, so I'll only be able to scratch the surface.


#### Short-read sequencing

Short-read sequencing (mostly synonymous with Illumina sequencing) has some clear advantages over long-read sequencing. It's often cheaper, especially if you have a large number of isolates to multiplex. It's also been in widespread use for longer, so there are a lot of mature tools and pipelines.

A lot can be learned from short reads alone. Is a gene present in the genome? What alleles are present? What species is the genome? What is the phylogeny of your samples? If you only need to answer questions like these, you may not need long reads.

The main downside of short-read sequencing regards assembly completeness. Since short reads cannot span many repeats in the genome, they do not provide enough information to get a complete assembly (one contig per replicon) in all but the very simplest bacterial genomes. A short-read assembly will therefore consist of fragmented contigs, some big and some small, making it difficult to see the large-scale genomic structure. How many plasmids are in the genome? Is a gene of interest on the chromosome or on a plasmid? Is a gene of interest in a mobilizable cluster? How are insertion sequences distributed in the genome? Is a phage sequence separate and circular or integrated into the chromosome? These kinds of questions can require long reads to answer.

Regarding sequencing depth, more is better! However, I don't see much benefit in exceeding 200× – beyond that is probably a waste of money. For financial reasons, much shallower sequencing is often called for, e.g. when it enables sequencing of more isolates (100 isolates at 20× depth could cost the same as 20 isolates at 100× depth). Shallow sequencing is fine for many downstream applications, but it will result in more fragmented assemblies. I wouldn't go below 20× depth unless you're sure that your analysis will be okay with it (and don't expect good assemblies).

For Illumina sequencing, there are a couple ways of preparing the library: either the [TruSeq prep](https://www.illumina.com/products/by-type/sequencing-kits/library-prep-kits/truseq-dna-pcr-free.html) which involves fragmenting the DNA mechanically or the [Nextera prep](https://www.illumina.com/products/by-type/sequencing-kits/library-prep-kits/nextera-dna-flex.html) (now confusingly called 'Illumina DNA Prep') which uses an enzyme to fragment the DNA. If you have the option, I'd recommend TruSeq as it results in more even depth of coverage. Nextera, in contrast, results in more variable depth because the enzyme prefers to cut some sequences more than others. If Nextera is your only option, that's okay, but I would encourage you to aim for deeper sequencing to compensate for the less-even coverage.


#### Long-read sequencing

Oxford Nanopore and PacBio both make long-read sequencing platforms. Their reads can be tens of kilobases in length (or more), which means that in most cases reads can span all repeats in a bacterial genome, enabling a complete assembly (one contig per replicon). You can then answer scientific questions about large-scale structure such as those listed above.

One downside of long-read sequencing is cost. While it has gotten cheaper over time, long-read sequencing is still more expensive than short-read sequencing in many cases. The major exception to this would be the initial cost of the sequencer itself: the Oxford Nanopore MinION is far cheaper than any Illumina sequencer.

Another downside is the error rate of the final assembly. Long reads have more errors than short reads, and while most of these errors can be fixed in the assembly/polishing steps, some can remain. A common case is homopolymer runs: a sequence like `TTTTTTTTTTT` is challenging for long-read sequencers/assemblers to get right (e.g. they might report 10 Ts instead of 11). This problem isn't as bad as it used to be, but it still exists.

I'm not very qualified to discuss Oxford Nanopore vs PacBio, as most of my experience is with the former, so I'll keep this brief. Oxford Nanopore is much cheaper if you need to buy the sequencer itself. Assuming you already have a sequencer, Oxford Nanopore is still probably cheaper on a per-bacterial-genome basis, though PacBio's newer platforms are not too far behind here. Oxford Nanopore can give better read lengths than PacBio (especially if you optimise for ultra-long reads) which will be handy if you have a very complex genome with long repeats. For a typical not-too-complex bacterial genome, both Oxford Nanopore and PacBio reads will be long enough. The Oxford Nanopore MinION is very small and portable, which is a big advantage for field work and mobile labs. Modern PacBio platforms can make HiFi reads which have much better sequence accuracy than Oxford Nanopore reads. PacBio is probably also better for consensus accuracy: the assembly will have fewer residual errors. For this reason, if you're after an extremely high-quality assembly, PacBio sequencing is a good choice. However, I think that hybrid sequencing (see [below](#hybrid-sequencing)) can yield the best genomes of all.

Regarding long-read sequencing depth, I'll say roughly the same thing as for short reads: more is better up to \~200×, and less than 20× is a bad idea. If you want very high quality assemblies, aim for the high end of that spectrum. If you're doing PacBio HiFi sequencing, then you can probably get away with the lower end of that spectrum. A decent MinION run should yield about 10 Gbp of total sequence, and a very good MinION run will yield about 20 Gbp. This means that multiplexing 12 or 24 genomes on a single run is often a good amount. However, getting an even distribution of barcodes can be difficult (e.g. you might get 2 Gbp from one barcode and only 200 Mbp from another) so be prepared for the possibility that you might need to re-sequence some of the genomes on your run.

Both Oxford Nanopore and PacBio are single-molecule sequencers: each read comes from one molecule of DNA. This means that your read lengths are largely dependent on the length of your DNA molecules. So if your DNA gets sheared during extraction/prep (bead-beating, lots of pipetting, freeze/thaw cycles, etc.) your reads will be shorter. Conversely, a gentle extraction/prep can yield very long reads. Some researchers have [designed protocols for ultra-long reads](https://www.nature.com/articles/nbt.4060), but this is probably not necessary for most bacterial genomes where repeats aren't as bad as in eukaryotes. However, longer is generally better, so I would recommend you aim for the longest reads you reasonably can. The one caveat to this regards small plasmids.

Small plasmids can be a problem with long-read sequencing. Consider a genome with a 3 kbp plasmid where DNA was extracted for Nanopore sequencing with a typical length of 20 kbp. This means that the 3 kbp plasmid is likely to be completely intact and still circular! A [ligation-based prep](https://store.nanoporetech.com/sample-prep/ligation-sequencing-kit.html) would then fail to sequence this plasmid because there are no blunt DNA ends on which adapters can be ligated. For this reason, small plasmids can be very underrepresented in long-read sets. Shearing your DNA might solve the problem, but it would result in shorter reads that could compromise the rest of your assembly. A better solution is to use a [rapid prep](https://store.nanoporetech.com/catalog/product/view/id/226/s/rapid-barcoding-kit/) instead, as that doesn't rely on blunt ends. Or you could do hybrid sequencing (see below) and rely on Illumina reads to capture the small plasmid. [We recently wrote a paper that explores this topic in more detail](https://doi.org/10.1099/mgen.0.000631), so take a look if you're interested.


#### Hybrid sequencing

Hybrid sequencing refers to getting both short and long reads from the sample isolate. This can give you the best of both worlds! Pretty much all downstream analyses will be available to you. Having a nice hybrid read set will also allow you to make a very high-quality assembly, perfect for creating reference genomes. It will also solve the small plasmid problem described above. Specifically, I would recommend you produce a deep (>100×) long-read set and a medium (\~50×) short-read set. This will allow you to perform a long-read-first hybrid assembly (e.g. Autocycler+Medaka+Polypolish+Pypolca), which in my experience is the most accurate.

I strongly encourage you to grow and extract the DNA only once for each isolate, using the same DNA for both short-read and long-read sequencing. This way you can be certain that the two read sets are in exact agreement and avoid the problems described in the [hybrid read set mismatch](#appendix-hybrid-read-set-mismatch) appendix.

<br>





<br>

## Appendix: Oxford Nanopore basecalling

One of the interesting things about Oxford Nanopore sequencing is that it is very dependent on the basecalling process: the conversion of raw signal reads to FASTQ reads. It's not a simple task, and improvements in basecalling algorithms have yielded huge improvements in recent years.

You have three main options for Oxford Nanopore basecalling:
* Let the MinKNOW sequencing software do basecalling for you. This is a great option if you're using a GPU-accelerated platform like the GridION, MinIT or MinION Mk1C. However, if you're limited to basecalling on a laptop's CPU, this could be quite slow.
* Use the command-line tool [Guppy](https://community.nanoporetech.com/downloads) to basecall your reads. You can sequence on a laptop, move the raw reads to a big computer (ideally with a GPU) and basecall them there. Guppy also has built-in demultiplexing and barcode trimming. This is what I use and what I'd recommend to most people who aren't sequencing on a GPU-accelerated platform.
* Use an experimental basecaller like [Bonito](https://github.com/nanoporetech/bonito). This might give you higher accuracy reads but will probably be more complicated. Only go this route if you like to experiment or stay on the bleeding edge!

If you're basecalling through MinKNOW or command-line Guppy, you'll be able to choose a basecalling model, and there is a trade-off between speed and accuracy. 'Fast' basecalling models are the quickest but yield less accurate reads. 'High accuracy' or 'hac' model are in the middle. And 'Super' or 'sup' models are slowest but give the highest accuracy reads. Higher accuracy reads also result in a higher accuracy consensus sequence, so it's absolutely worth doing the best basecalling you can! Aim for a super model, if you have the computational resources to pull it off.

Regardless of which option you choose, keep your software up-to-date! Basecalling continues to improve, so you'll want the latest version of your basecaller. If you're analysing older data, it might be worth re-basecalling the reads with a new basecaller version to get better accuracy, so don't delete your raw reads!

<br>





<br>

## Appendix: hybrid read set mismatch

Imagine the following scenario. You've done Illumina sequencing on a bacterial isolate, and then weeks later decide it would be nice to have long reads for that isolate as well. So you pull it out of the freezer, grow it, extract DNA and sequence using a MinION. Now you have a hybrid read set! But what if you didn't actually sequence quite the same genome? Maybe there was a plasmid in the first read set which was lost in the second read set. Or maybe a transposon moved. Or maybe your original sample was mixed and you actually sequenced two completely different genomes. Now you have two read sets for two genomes (maybe slightly different, maybe very different) and hybrid assembly methods might stumble and fall over.

To avoid this problem, it's best to use a single DNA extraction for both read sets (as I recommend [above](#hybrid-sequencing)). If that's not possible, then you should at least keep this possible complication in the back of your mind.

If your read sets are from different DNA extractions and your hybrid assembly isn't working, you'll need to investigate. I'd suggest doing both a short-read-only and long-read-only assembly for your isolate and comparing the results. If there are major differences between your read sets (i.e. they are of totally different genomes) then it should be obvious! More subtle differences are harder. Good luck :sweat_smile:

<br>





<br>

## Appendix: gaps in my knowledge

There are many aspects of bacterial whole-genome assembly where I have little to no experience. Here are some of my known unknowns ([as Donald Rumsfeld would say](https://en.wikipedia.org/wiki/There_are_known_knowns)):
* [BGI sequencing platforms](https://www.bgi.com/global/resources/sequencing-platforms). Like Illumina platforms, these generate short reads.
* Modern PacBio platforms like the [Sequel II/IIe](https://www.pacb.com/products-and-services/sequel-system). The last time I used PacBio reads for a microbial genome, they were from a PacBio RSII. I also don't have experience with PacBio [CCS/HiFi reads](https://www.pacb.com/smrt-science/smrt-sequencing/hifi-reads-for-highly-accurate-long-read-sequencing) (I've only used PacBio [CLR reads](https://www.pacb.com/wp-content/uploads/2015/09/Pacific-Biosciences-Glossary-of-Terms.pdf)) or multiplexing on a PacBio sequencer. Thankfully, [Dan Browne](https://twitter.com/DRBFX) had a chat with me to fill me in on some details, so I now have second-hand knowledge here. Thanks, Dan!
* Synthetic long reads, such as the [Morphoseq approach](https://longastech.com/). These use a combination of lab and computational methods to produce long reads from a short-read sequencer. They can then be assembled as if they came from a long-read sequencer.
* [Mate pair sequencing](https://sapac.illumina.com/science/technology/next-generation-sequencing/mate-pair-sequencing.html) where Illumina reads are produced from long-insert libraries, providing more information for assembly. While this seems common for big eukaryote genomes, I haven't encountered it for microbial genomes.

If you've used any of these for bacterial whole-genome sequencing and assembly, I'd be very keen to hear your experiences. And if you spot any unknown unknowns, let me know about them too! I can then update this guide accordingly.

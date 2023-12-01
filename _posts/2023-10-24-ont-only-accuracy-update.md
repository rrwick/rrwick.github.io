---
layout: post
title:  "ONT-only accuracy: 5 kHz and Dorado"
date:   2023-10-24
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10038672.svg)](https://doi.org/10.5281/zenodo.10038672)

About six months ago, I tested ONT-only assembly accuracy using R10.4.1 reads:<br>[ONT-only accuracy with R10.4.1](https://rrwick.github.io/2023/05/05/ont-only-accuracy-with-r10.4.1.html).

In that post, I found that while the median ONT-only accuracy was quite good (~Q50), it varied a lot between bacterial species. Some genomes (_Salmonella_ and _Listeria_) had very few errors, while others (_E. coli_ and _Campylobacter_) had many. DNA methylation motifs seemed to be the biggest cause of errors, presumably because the basecaller wasn't trained on native DNA that included those methylation patterns.

There have been a couple of ONT developments since that post: the move to a 5&nbsp;kHz sampling rate[^khz] and the maturation of [Dorado](https://github.com/nanoporetech/dorado)[^dorado]. So I decided it was time to once again quantify ONT-only assembly accuracy on bacterial genomes.




## Methods

I used the following nine genomes, sequenced by my colleagues [Louise](https://twitter.com/JuddLmj) and [Hasini](https://www.linkedin.com/in/hasini-walpola-7a6045a7):
* _Campylobacter jejuni_ ([ATCC-33560](https://www.atcc.org/products/33560))
* _Campylobacter lari_ ([ATCC-35221](https://www.atcc.org/products/35221))
* _Escherichia coli_ ([ATCC-25922](https://www.atcc.org/products/25922))
* _Listeria ivanovii_ ([ATCC-19119](https://www.atcc.org/products/19119))
* _Listeria monocytogenes_ ([ATCC-BAA-679](https://www.atcc.org/products/baa-679))
* _Listeria welshimeri_ ([ATCC-35897](https://www.atcc.org/products/35897))
* _Salmonella enterica_ ([ATCC-10708](https://www.atcc.org/products/10708))
* _Vibrio cholerae_ ([ATCC-14035](https://www.atcc.org/products/14035))
* _Vibrio parahaemolyticus_ ([ATCC-17802](https://www.atcc.org/products/17802))

These include the same five genomes I used [last time](https://rrwick.github.io/2023/05/05/ont-only-accuracy-with-r10.4.1.html) plus four additional ones. Each had deep ONT and Illumina reads, and I produced ground-truth genomes using my usual approach[^perfect].

Unlike last time where I only tested sup basecalling, this time I tested a range of basecalling models:
* `dna_r10.4.1_e8.2_400bps_fast@v4.2.0`
* `dna_r10.4.1_e8.2_400bps_hac@v4.2.0`
* `dna_r10.4.1_e8.2_400bps_sup@v4.2.0`
* `res_dna_r10.4.1_e8.2_400bps_sup@2023-09-22_bacterial-methylation`

For brevity, I'll refer to these as 'fast', 'hac', 'sup' and 'res'. The first three are the current standard [Dorado models](https://github.com/nanoporetech/dorado#dna-models) at different levels of speed/accuracy[^levels]. The last one is interesting – it's a sup-sized research model available on [Rerio](https://github.com/nanoporetech/rerio) that has been fine-tuned for native bacterial DNA[^res]. All basecalling was simplex only, and I demultiplexed and trimmed using Dorado. I then did a bit of read QC with [Filtlong](https://github.com/rrwick/Filtlong): first `--min_length 10000` to remove short reads[^plasmids] then `--keep_percent 90` to discard the worst 10% of each read set. The post-QC read sets were nice and deep (ranging from about 1&nbsp;Gbp to 2.5&nbsp;Gbp per genome).

I made a [Trycycler](https://github.com/rrwick/Trycycler/wiki) ONT-only assembly from each read set, also trying [Medaka](https://github.com/nanoporetech/medaka) when possible[^medaka]. This resulted in seven assemblies for each genome: fast, hac, hac+Medaka, sup, sup+Medaka, res and res+Medaka. To quantify read accuracy, I aligned the ONT reads to my ground-truth genomes[^acc] and calculated identity from all alignments >10&nbsp;kbp in length. To quantify assembly accuracy, I counted the number of differences[^diff] between each assembly and my ground-truth genome.




## Results: read accuracy

This table shows the simplex read identity (top) and qscore[^qscore] (bottom) across all nine genomes:

|   average   |      fast      |      hac       |      sup       |      res       |
|:-----------:|:--------------:|:--------------:|:--------------:|:--------------:|
| mean        | 91.0%<br>Q10.5 | 96.2%<br>Q14.1 | 97.1%<br>Q15.4 | 97.3%<br>Q15.7 |
| median      | 92.1%<br>Q11.0 | 97.6%<br>Q16.2 | 98.6%<br>Q18.5 | 98.7%<br>Q19.0 |
| mode[^mode] | 93.0%<br>Q11.5 | 98.2%<br>Q17.4 | 99.0%<br>Q20.0 | 99.2%<br>Q21.0 |




## Results: assembly accuracy

This table shows the error count (top) and qscore (bottom) for each assembly:

| Genome                      | fast               | hac              | hac+<br>Medaka   | sup              | sup+<br>Medaka   | res             | res+<br>Medaka   |
|:---------------------------:|:------------------:|:----------------:|:----------------:|:----------------:|:----------------:|:---------------:|:----------------:|
| _Campylobacter jejuni_      |    2539<br>Q28.4   |   159<br>Q40.5   |   137<br>Q41.1   |    27<br>Q48.2   |    73<br>Q43.8   |   13<br>Q51.3   |    23<br>Q48.9   |
| _Campylobacter lari_        |    2739<br>Q27.4   |   208<br>Q38.6   |   139<br>Q40.4   |    28<br>Q47.3   |    65<br>Q43.7   |   18<br>Q49.2   |    17<br>Q49.5   |
| _Escherichia coli_          |    5510<br>Q29.8   |   110<br>Q46.8   |    96<br>Q47.3   |    70<br>Q48.7   |    72<br>Q48.6   |   46<br>Q50.5   |    44<br>Q50.7   |
| _Listeria ivanovii_         |     936<br>Q34.9   |     9<br>Q55.1   |     6<br>Q56.9   |     9<br>Q55.1   |     4<br>Q58.6   |    8<br>Q55.6   |     5<br>Q57.7   |
| _Listeria monocytogenes_    |     795<br>Q35.7   |     2<br>Q61.7   |     0<br>Q∞      |     0<br>Q∞      |     0<br>Q∞      |    0<br>Q∞      |     0<br>Q∞      |
| _Listeria welshimeri_       |     668<br>Q36.2   |     4<br>Q58.5   |     1<br>Q64.5   |     2<br>Q61.5   |     3<br>Q59.7   |    2<br>Q61.5   |     1<br>Q64.5   |
| _Salmonella enterica_       |    4132<br>Q30.7   |    45<br>Q50.3   |    35<br>Q51.4   |     8<br>Q57.8   |    13<br>Q55.7   |    5<br>Q59.8   |     7<br>Q58.4   |
| _Vibrio cholerae_           |    4664<br>Q29.5   |    50<br>Q49.2   |    24<br>Q52.4   |    22<br>Q52.7   |    14<br>Q54.7   |    0<br>Q∞      |     4<br>Q60.2   |
| _Vibrio parahaemolyticus_   |    4202<br>Q30.9   |    42<br>Q50.9   |     7<br>Q58.7   |     7<br>Q58.7   |    17<br>Q54.8   |    7<br>Q58.7   |     7<br>Q58.7   |
| **total/average**[^average] | **26185<br>Q30.8** | **629<br>Q47.0** | **445<br>Q48.5** | **173<br>Q52.6** | **261<br>Q50.8** | **99<br>Q55.0** | **108<br>Q54.6** |




## Discussion and conclusions

As expected, the fast reads/assemblies were pretty rough and had a lot of errors, the hac reads/assemblies were much better, and the sup reads/assemblies were the best. You should really only use fast basecalling if you're computationally constrained or doing an analysis that isn't sensitive to errors, e.g. species identification.

Overall, assembly accuracy definitely improved compared to the [last time](https://rrwick.github.io/2023/05/05/ont-only-accuracy-with-r10.4.1.html) I tested it. The current hac accuracy is similar to my previous sup accuracy, and the current sup accuracy is very good: less than 100 errors for all genomes and less than 10 errors for most. There are still some _E.&nbsp;coli_ errors in [M1.EcoMI methylation motifs](https://doi.org/10.1128/mBio.01602-15) and _Campylobacter_ errors in [CtsM methylation motifs](https://doi.org/10.1073/pnas.1703331114), and long homopolymers (e.g. 10+&nbsp;bp) sometimes have indel errors. So ONT accuracy still struggles with the same things it used to, but it's moving in the right direction.

Regarding Medaka, I've been less impressed by its polishing than I used to be. It clearly helped with the hac assemblies, but it performed erratically with the sup/res assemblies, making things worse more often than better. My recommendation is therefore to use Medaka if you are assembling hac reads but skip it for sup/res reads.

Going into this, I was most curious about how [Rerio](https://github.com/nanoporetech/rerio)'s research bacterial basecalling model would perform, and it did well! Read accuracy was slightly improved over sup, and assembly accuracy was the same or better than sup for all nine genomes. At the time of writing, this seems to be the best basecalling model for native bacterial DNA. ONT has been trying to simplify their sequencing/analysis lately[^choices], so I understand that they are reluctant to provide additional basecalling models in Dorado. But I hope they continue to host organism-specific fine-tuned models on Rerio, for users who want to get the most out of ONT-only sequencing.

Overall, I'm very happy with these results – they show a big accuracy improvement since earlier this year! For bacterial genomes, near-perfect ONT-only assemblies are now often possible, and truly perfect ONT-only assemblies are occasionally possible.




## Read availability

**1 Dec 2023 update:** I got a number of requests for the FASTQs, and I'm happy to announce that they are now available on NCBI: [PRJNA1042815](https://www.ncbi.nlm.nih.gov/bioproject/PRJNA1042815).

Note that each isolate has multiple associated SRA runs, so check the library name to make sure you're getting the one you want. The runs used in this blog post contain the following text in their library name: **2023&#8209;09_illumina**, **2023&#8209;09_nanopore_fast**, **2023&#8209;09_nanopore_hac**, **2023&#8209;09_nanopore_sup** or **2023&#8209;09_nanopore_res**.

Also note that these are pre-QC reads, so many of the read sets are very large and have a poor N50. You therefore might want to do some read-length-based QC after downloading.




## Footnotes

[^khz]: Previously, ONT sequencing generated raw data at a 4&nbsp;kHz rate (~10 samples/nucleotide). Now they use 5&nbsp;kHz sampling (~12.5 samples/nucleotide), giving basecallers a bit more data to work with.

[^dorado]: While Dorado has been available since 2022, it has only recently gotten to the point where it's a full replacement for Guppy. It's now the default basecaller in MinKNOW and can do barcode demultiplexing/trimming.

[^perfect]: [Trycycler](https://github.com/rrwick/Trycycler/wiki), [Polypolish](https://github.com/rrwick/Polypolish), [POLCA](https://github.com/alekseyzimin/masurca) and manual curation. See [this tutorial](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki) for more details.

[^levels]: The `fast` model is small: high speed and low accuracy. The `hac` model is medium: slower speed and higher accuracy. And the `sup` model is big: slowest speed and highest accuracy.

[^res]: If you have access to the ONT community site, you can read more about it here: [community.nanoporetech.com/posts/research-release-basecall](https://community.nanoporetech.com/posts/research-release-basecall).

[^plasmids]: Discarding all reads <10&nbsp;kbp in size is quite aggressive. I wouldn't normally do this, but the unfiltered read sets were very deep so I could get away with it here. It makes assembly easier, but it also pretty much guarantees that small plasmids will be lost.

[^medaka]: There are [Medaka models](https://github.com/nanoporetech/medaka/tree/master/medaka/data) for hac and sup reads (`r1041_e82_400bps_hac_v4.2.0` and `r1041_e82_400bps_sup_v4.2.0`). I also used the sup Medaka model for my res reads, since that basecalling model is based on sup. There isn't a fast-read Medaka model, so I didn't use Medaka on my fast-read assemblies. 

[^acc]: Read accuracies were calculated using _all_ ONT reads, i.e. before I ran [Filtlong](https://github.com/rrwick/Filtlong). This means there was no read QC other than barcode demultiplexing (which serves as a bit of QC because very bad reads are more likely to end up in the unclassified bin). You can therefore expect higher mean and median values if you do some read QC (e.g. discarding 'fail' reads or running Filtlong).

[^diff]: I used my [`compare_assemblies.py`](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki/Comparing-assemblies) script to get a difference count.

[^mode]: The modal accuracy represents the peak of the read-identity distribution. To calculate it, I rounded each identity value to three decimal places (e.g. 0.972316 → 0.972) and took the most common value.

[^qscore]: Qscore is defined as -10 × log<sub>10</sub>(errors / genome size). I think of it like this: the qscore tens place is the number of nines in the accuracy: Q20 = 99%, Q30 = 99.9%, Q40 = 99.99%, etc.

[^average]: I calculated average qscores using the total number of errors across all nine genomes divided by the total size of all nine genomes.

[^choices]: ONT sequencing has had a lot of choices in recent years: R9.4.1 vs R10.4.1, 260&nbsp;bp/s vs 400&nbsp;bp/s, fast vs hac vs sup, rapid vs ligation, Guppy vs Dorado, simplex vs duplex, fast5 vs pod5, etc. Some of these options are being retired (R9.4.1, 260&nbsp;bp/s, Guppy and fast5), which will hopefully simplify things going forward.

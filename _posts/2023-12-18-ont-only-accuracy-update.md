---
layout: post
title:  "Yet another ONT accuracy test: Dorado v0.5.0"
date:   2023-12-18
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.10397817.svg)](https://doi.org/10.5281/zenodo.10397817)


This is my third ONT-only accuracy update in 2023 – it's been a busy year for ONT! This one is motivated by the early-December release of [Dorado v0.5.0](https://github.com/nanoporetech/dorado/releases/tag/v0.5.0) and its new v4.3 basecalling models.

[Last time](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html), I tested the three standard Dorado models (fast, hac and sup) along with a research model on [Rerio](https://github.com/nanoporetech/rerio) which was fine-tuned for bacteria. However, that research model now seems to be obsolete, as the new v4.3 basecalling models incorporate this bacterial training[^bac]. This time, I only tested the sup model (not fast or hac), both to save myself time and also because I think of sup-basecalling as the default choice. Given sufficient compute (which we have with [Onion](https://rrwick.github.io/2023/12/09/ont-desktop.html)), why would you use anything but sup?




## Methods

I used the same genomes and methods as [my last accuracy post](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html), so look there for the details. The results below compare the previous `dna_r10.4.1_e8.2_400bps_sup@v4.2.0` model to the current `dna_r10.4.1_e8.2_400bps_sup@v4.3.0` model.




## Results: read accuracy

This table shows the simplex read identity (left) and qscore (right) across all nine genomes:

|   average   |   sup v4.2.0   |   sup v4.3.0   |
|:-----------:|:--------------:|:--------------:|
| mean        |  97.1%, Q15.4  |   97.7%, Q16.4 |
| median      |  98.6%, Q18.5  |   99.1%, Q20.5 |
| mode        |  99.0%, Q20.0  |   99.4%, Q22.2 |
 



## Results: assembly accuracy

This table shows the error count (left) and qscore (right) for each assembly:

| Genome                      | sup v4.2.0       | sup v4.3.0      |
|:---------------------------:|:----------------:|:---------------:|
| _Campylobacter jejuni_      |     27, Q48.2    |     5, Q55.5    |
| _Campylobacter lari_        |     28, Q47.3    |    18, Q49.2    |
| _Escherichia coli_          |     70, Q48.7    |     1, Q67.2    |
| _Listeria ivanovii_         |      9, Q55.1    |     5, Q57.7    |
| _Listeria monocytogenes_    |      0, Q∞       |     0, Q∞       |
| _Listeria welshimeri_       |      2, Q61.5    |     1, Q64.5    |
| _Salmonella enterica_       |      8, Q57.8    |     3, Q62.0    |
| _Vibrio cholerae_           |     22, Q52.7    |     2, Q63.2    |
| _Vibrio parahaemolyticus_   |      7, Q58.7    |     2, Q64.1    |
| **total, average**           |  **173, Q52.6**  |  **37, Q59.3**  |


Since there are only 37 remaining errors in the sup v4.3.0 assemblies, I've included them all below. Each comparison shows the ONT-only assembly (top) vs the Illumina-polished reference (bottom):
<details>
<summary><b>All sup v4.3.0 errors</b></summary>
<figure>
<pre>
<code>ATCC_33560_Campylobacter_jejuni
-------------------------------
  chromosome 148667-148706: TAAATTTAAGTAAAAATTTA--TTTTTTTTTTTATAATTCTA
  chromosome 148667-148708: TAAATTTAAGTAAAAATTTATTTTTTTTTTTTTATAATTCTA
                                                **                    

chromosome 1493751-1493791: TAAAAAAAGGCATAATGCCTAAAATCAAAAACATAAAAATA
chromosome 1493753-1493793: TAAAAAAAGGCATAATGCCTGAAATCAAAAACATAAAAATA
                                                *                    

chromosome 1576231-1576271: TTACCAGATAATGAAAATTTTGGGGTTTTTTCATGAAAAAT
chromosome 1576233-1576273: TTACCAGATAATGAAAATTTCGGGGTTTTTTCATGAAAAAT
                                                *                    

chromosome 1764130-1764170: TTATTATCAAAATTTATCCCAAAGTTGTCTAATACAAATTC
chromosome 1764132-1764172: TTATTATCAAAATTTATCCCGAAGTTGTCTAATACAAATTC
                                                *                    


ATCC_35221_Campylobacter_lari
-----------------------------
  chromosome 284330-284370: ATGGAGTTCAAACCTTTGATTGAAAGACAAGATAAGATTGT
  chromosome 284330-284370: ATGGAGTTCAAACCTTTGATCGAAAGACAAGATAAGATTGT
                                                *                    

  chromosome 485386-485429: AAGGACTCCCCAAAAGCATTAATTGAAATAGGAAATATTCCTCA
  chromosome 485386-485429: AAGGACTCCCCAAAAGCATTGATCGAAATAGGAAATATTCCTCA
                                                *  *                    

  chromosome 491960-491999: AAACCAATATACCCCCCCCC-TTTTTTTTTCAATCATAAAT
  chromosome 491960-492000: AAACCAATATACCCCCCCCCTTTTTTTTTTCAATCATAAAT
                                                *                    

  chromosome 521071-521111: TTCATTGGTTCCTTCTACCAAATCTTTTGCGTCTTGATTTA
  chromosome 521072-521112: TTCATTGGTTCCTTCTACCAGATCTTTTGCGTCTTGATTTA
                                                *                    

  chromosome 527347-527387: CTTCTTCTTCAAATTCTTTAAATCCATCAAATACAACTATG
  chromosome 527348-527388: CTTCTTCTTCAAATTCTTTAGATCCATCAAATACAACTATG
                                                *                    

  chromosome 548148-548187: TAAAGTCATAATTTTTTCCA---TTTTTTTTTTTTTTTTGAAA
  chromosome 548149-548191: TAAAGTCATAATTTTTTCCATTTTTTTTTTTTTTTTTTTGAAA
                                                ***                    

  chromosome 568530-568570: AAATATTGTGTTGATTTTACAATCAATGAAACCCAACTTTT
  chromosome 568534-568574: AAATATTGTGTTGATTTTACGATCAATGAAACCCAACTTTT
                                                *                    

  chromosome 693670-693710: GTTTGATGACATGGTGGCTAAATCTGTGCCTTTTTATGCGC
  chromosome 693674-693714: GTTTGATGACATGGTGGCTAGATCTGTGCCTTTTTATGCGC
                                                *                    

  chromosome 755888-755928: AAGGTGGTTTTATCATGGATTTATGTTTCGCTATGAAAGGC
  chromosome 755892-755932: AAGGTGGTTTTATCATGGATCTATGTTTCGCTATGAAAGGC
                                                *                    

  chromosome 871342-871382: TCAGCCTTGTTCTGCCTGATTGGATATCTCCCAAGCCACGC
  chromosome 871346-871386: TCAGCCTTGTTCTGCCTGATCGGATATCTCCCAAGCCACGC
                                                *                    

  chromosome 889695-889735: GAACCAAAGGAACACGGGTAAATCCACCCACCATTACAATT
  chromosome 889699-889739: GAACCAAAGGAACACGGGTAGATCCACCCACCATTACAATT
                                                *                    

  chromosome 924953-924993: TAGCTGAGCCTTGAAATTTAAATCCATCATAAGAAAATGTT
  chromosome 924957-924997: TAGCTGAGCCTTGAAATTTAGATCCATCATAAGAAAATGTT
                                                *                    

chromosome 1045770-1045810: TTTTAATGAATTTGCTGGATTGAAATTGATTGGATCCATTG
chromosome 1045774-1045814: TTTTAATGAATTTGCTGGATCGAAATTGATTGGATCCATTG
                                                *                    

chromosome 1052250-1052290: TCATTTTTATGAGAAAATGATTCTTTTTAAAATAGCATTGA
chromosome 1052254-1052293: TCATTTTTATGAGAAAATGA-TCTTTTTAAAATAGCATTGA
                                                *                    

chromosome 1359682-1359722: TCATCCATCAAACGATTGATTGCCATTGCAAAATTTCTTGC
chromosome 1359685-1359725: TCATCCATCAAACGATTGATCGCCATTGCAAAATTTCTTGC
                                                *                    


ATCC_25922_Escherichia_coli
---------------------------
  chromosome 864422-864461: CAGGGGGGGATGCTCATTCT-GGGGGGGAGAAAAAAGATGG
  chromosome 864422-864462: CAGGGGGGGATGCTCATTCTGGGGGGGGAGAAAAAAGATGG
                                                *                    


ATCC_19119_Listeria_ivanovii
----------------------------
  chromosome 808062-808101: AGATGGTAAAGCTTCTCATG-TTTTTTTTTTTATACCGCGC
  chromosome 808062-808102: AGATGGTAAAGCTTCTCATGTTTTTTTTTTTTATACCGCGC
                                                *                    

chromosome 1127956-1127995: GACTGGATGGCTTTTTGTCA-TTTTTTTTTTGAAGGCGCAA
chromosome 1127957-1127997: GACTGGATGGCTTTTTGTCATTTTTTTTTTTGAAGGCGCAA
                                                *                    

chromosome 2671863-2671902: ATAAGAATTTTACCCCACAC---AAAAAAAAAAAAAAAAAACC
chromosome 2671865-2671907: ATAAGAATTTTACCCCACACAAAAAAAAAAAAAAAAAAAAACC
                                                ***                    


ATCC_35897_Listeria_welshimeri
------------------------------
  chromosome 918445-918484: CCAAGCCGCGGCGCTTGGTC-TTTTTTTTACGCGTTTAGTT
  chromosome 918445-918485: CCAAGCCGCGGCGCTTGGTCTTTTTTTTTACGCGTTTAGTT
                                                *                    


ATCC_10708_Salmonella_enterica
------------------------------
  chromosome 665795-665834: AAGGGTGAGAGGGGATCTCT-CCCCCTCTGATTGGCTGTTA
  chromosome 665795-665835: AAGGGTGAGAGGGGATCTCTCCCCCCTCTGATTGGCTGTTA
                                                *                    

chromosome 2716525-2716564: TAAGGAACGCCATGAAAAAG-TTTTTTTTTGCCGCTGCGCT
chromosome 2716526-2716566: TAAGGAACGCCATGAAAAAGTTTTTTTTTTGCCGCTGCGCT
                                                *                    

       plasmid 12128-12167: AATGGCAAACGTCTCACTGG-CCCCCCCCCCCCCCCCCCCC
       plasmid 12128-12168: AATGGCAAACGTCTCACTGGCCCCCCCCCCCCCCCCCCCCC
                                                *                    


ATCC_14035_Vibrio_cholerae
--------------------------
chromosome_1 1469609-1469649: GTGCCTGTACTTGGTTGGATTGCCACACCCCATGCACTACC
chromosome_1 1469609-1469649: GTGCCTGTACTTGGTTGGATCGCCACACCCCATGCACTACC
                                                  *                    

chromosome_1 1475125-1475164: AATTCTATGAAGGTGCAGAG-AAAAAAAAGAGGAAATGCGT
chromosome_1 1475125-1475165: AATTCTATGAAGGTGCAGAGAAAAAAAAAGAGGAAATGCGT
                                                  *                    


ATCC_17802_Vibrio_parahaemolyticus
----------------------------------
  chromosome_1 970379-970418: TATCAACGAATTTTAGATAC-AAAAAAAAAGCGTAACGGCC
  chromosome_1 970379-970419: TATCAACGAATTTTAGATACAAAAAAAAAAGCGTAACGGCC
                                                  *                    

  chromosome_2 960683-960722: ACGTAGTTTACAATTTAAGC-AAAAAAAAAGAGGGCATAAT
  chromosome_2 960683-960723: ACGTAGTTTACAATTTAAGCAAAAAAAAAAGAGGGCATAAT
                                                  *                    
</code>
</pre>
</figure>
</details>




## Discussion and conclusions

The v4.3.0 model is clearly a big improvement over v4.2.0! Read accuracy got noticeably better: v4.3.0 reads had roughly 2/3 of the errors compared to v4.2.0 reads. And there was an even bigger improvement in consensus accuracy: v4.3.0 assemblies had less than 1/4 of the errors compared to v4.2.0 assemblies. The remaining assembly errors are mostly in long homopolymers. The exception was _Campylobacter lari_ which had more errors than the other genomes, mostly in the `GATC` motif.[^gatc] It's clear that sup v4.3.0 is now the best basecalling model – much better than the bacterial research model on Rerio.

While there is also a [new version of Medaka](https://github.com/nanoporetech/medaka/releases/tag/v1.11.3) with polishing models to match the v4.3.0 basecalling models, it once again failed to help much. Three of the nine genomes got better with Medaka polishing, three got worse and three didn't change.[^medakaerrors] So my opinion on Medaka polishing of sup assemblies remains the same: don't bother.[^medakahac]

I should again emphasise that these ONT-only genomes came from very deep read sets and careful [Trycycler](https://github.com/rrwick/Trycycler/wiki) assembly, so a more typical ONT-only genome (e.g. 50–100× depth assembled with Flye) probably won't be quite this good. Another caveat is these genomes are from well-studied taxa, so their DNA modifications/motifs are more likely to be represented in ONT's basecaller training set.[^training] How well does ONT perform on _Rubeoparvulum_, _Abyssicoccus_, _Zhihengliuella_ and _Haloactinopolyspora_?[^obscure] It would be interesting to assess ONT-only accuracy on a _really_ diverse set of genomes.

Overall, I'm very impressed with these results, and perfect ONT-only bacterial assemblies are now looking closer than ever. And as always, there are plenty of developments on the horizon which may improve accuracy even further. The closest is the E8.2.1 motor protein, scheduled for release next year, which ONT claims will increase read accuracy by another couple qscores.[^motor] The steady year-after-year improvement of ONT sequencing, while occasionally frustrating (when I need to rebasecall all my data), has been very fun to witness.




## Footnotes

[^bac]: As reported by [Chris Seymour](https://twitter.com/iiSeymour/status/1732136835957080305) and [Mike Vella](https://twitter.com/vellamike/status/1732157858005782670).

[^gatc]: I haven't checked, but I assume this is due to Dam methylation, which occurs in many species, [including Campylobacter](https://www.pnas.org/doi/10.1073/pnas.1703331114).


[^medakaerrors]: The sup v4.3.0 + Medaka error counts were: 3, 28, 1, 4, 0, 1, 6, 3 and 1 (same genome order as the above table) for a total of 47 errors.

[^medakahac]: While I didn't test the hac v4.3.0 model this time, I [previously found](https://rrwick.github.io/2023/10/24/ont-only-accuracy-update.html) that Medaka _is_ quite beneficial for hac assemblies.

[^training]: I don't exactly know what's in ONT's basecaller training set. Presumably there's amplified DNA (no mods) and native DNA from humans, plants and various bacterial species. Perhaps also synthetic DNA with randomly placed modifications? And what else? I've asked ONT in the past, but they seem tight-lipped about it. Inquiring minds want to know!

[^obscure]: I've never heard of these taxa either. I just went to [GTDB](https://gtdb.ecogenomic.org) and grabbed some random obscure ones.

[^motor]: They talked about this in their [recent tech update presentation](https://nanoporetech.com/about-us/news/oxford-nanopore-announces-breakthrough-performance-simplex-single-molecule-accuracy). At T=20min, Lakmal talks about motor proteins and E8.2.1. At T=42min, Rosemary talks about the rollout, including the lot number which will indicate whether a kit contains E8.2.1.

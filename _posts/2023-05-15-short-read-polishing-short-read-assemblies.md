---
layout: post
title:  "Short-read polishing of short-read assemblies"
date:   2023-05-15
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7935999.svg)](https://doi.org/10.5281/zenodo.7935999)


[Marit Hetland](https://twitter.com/genomarit) recently asked me what I recommend for polishing short-read assemblies. [Unicycler](https://github.com/rrwick/Unicycler) used to run [Pilon](https://github.com/broadinstitute/pilon) at the end of its pipeline, but I saw many cases where it introduced errors, so I removed Pilon from Unicycler in a [recent version](https://github.com/rrwick/Unicycler/releases/tag/v0.5.0). I developed [Polypolish](https://github.com/rrwick/Polypolish) for short-read polishing of _long_-read assemblies and [tested it along with other polishers](https://doi.org/10.1371/journal.pcbi.1009802), but I don't really know how these tools perform on _short_-read assemblies.

So I didn't have a good answer for Marit, and her question made me realise that some tests were in order. Hence this blog post!



## Methods

I used the same five ATCC genomes described in my [previous post](https://rrwick.github.io/2023/05/05/ont-only-accuracy-with-r10.4.1.html) because I had both deep reads[^depth] and carefully curated ground-truth assemblies. I used Unicycler (which runs [SPAdes](https://github.com/ablab/spades)) to make the short-read assemblies. For polishers, I tried [POLCA](https://github.com/alekseyzimin/masurca) and Polypolish (my two favourites), as well as Pilon (formerly part of Unicycler).

Counting errors in short-read assemblies can be tricky due to their fragmentation. It's straightforward with single-copy contigs (multiplicity=1): just align the contig to the reference genome[^double] and count discrepancies[^count]. But repeat contigs (multiplicity>1) can be trickier, as they might have different error counts depending on which repeat instance they are compared to.

I decided to count the errors in each contig's _best_ alignment to the ground-truth reference. For example, if a contig represents a sequence repeated five times in the genome, and it matches at least one of those instances perfectly, I consider it error-free, even if there are errors in its alignments to the other four instances.

<details>
<summary>Here is the <code>count_errors.py</code> script I used:</summary>

{% highlight python %}
#!/usr/bin/env python3

# This script takes a PAF filename as input and outputs the total contig
# length, the error count and the corresponding qscore.

import collections
import math
import re
import sys


def main():
    contig_lengths = {}
    errors_per_contig = collections.defaultdict(lambda: float('inf'))
    with open(sys.argv[1], 'rt') as f:
        for paf_line in f:
            contig_name, contig_len, errors = get_name_len_errors(paf_line)
            contig_lengths[contig_name] = contig_len
            if errors < errors_per_contig[contig_name]:
                errors_per_contig[contig_name] = errors
    total_length = sum(contig_lengths.values())
    total_errors = sum(errors_per_contig.values())
    if total_errors == 0:
        qscore = 'inf'
    else:
        qscore = -10.0 * math.log10(total_errors / total_length)
        qscore = f'{qscore:.2f}'
    print(total_length, total_errors, qscore)


def get_name_len_errors(paf_line):
    parts = paf_line.rstrip('\n').split('\t')
    contig_name = parts[0]
    contig_len = int(parts[1])
    contig_start = int(parts[2])
    contig_end = int(parts[3])
    errors = contig_len - contig_end + contig_start
    for part in parts:
        if part.startswith('cg:Z:'):
            cigar = part[5:]
    errors += count_errors_in_cigar(cigar)
    return contig_name, contig_len, errors


def count_errors_in_cigar(cigar):
    errors = 0
    for p in re.findall(r'\d+[IDX=]', cigar):
        if p[-1] == 'X' or p[-1] == 'I' or p[-1] == 'D':
            errors += int(p[:-1])
    return errors


if __name__ == '__main__':
    main()
{% endhighlight %}

</details>

<details>
<summary>And here are the Bash commands I ran:</summary>

{% highlight bash %}
# Unicycler short-read assemblies
for d in ATCC_10708_Salmonella_enterica ATCC_17802_Vibrio_parahaemolyticus ATCC_25922_Escherichia_coli ATCC_33560_Campylobacter_jejuni ATCC_BAA-679_Listeria_monocytogenes; do
    cd ~/ATCC_assemblies/"$d"
    unicycler -1 reads_qc/illumina_1.fastq.gz -2 reads_qc/illumina_2.fastq.gz -o unicycler_illumina_only --threads 32
    cp unicycler_illumina_only/assembly.fasta unicycler.fasta
done

# Polishing
for g in ATCC_10708_Salmonella_enterica ATCC_17802_Vibrio_parahaemolyticus ATCC_25922_Escherichia_coli ATCC_33560_Campylobacter_jejuni ATCC_BAA-679_Listeria_monocytogenes; do
    cd ~/ATCC_assemblies/"$g"

    # Polypolish
    bwa index unicycler.fasta
    bwa mem -t 32 -a unicycler.fasta reads_qc/illumina_1.fastq.gz > alignments_1.sam
    bwa mem -t 32 -a unicycler.fasta reads_qc/illumina_2.fastq.gz > alignments_2.sam
    polypolish_insert_filter --in1 alignments_1.sam --in2 alignments_2.sam --out1 filtered_1.sam --out2 filtered_2.sam
    polypolish unicycler.fasta filtered_1.sam filtered_2.sam > unicycler_polypolish.fasta
    rm *.sam *.bwt *.pac *.ann *.amb *.sa

    # Pilon
    bwa index unicycler.fasta
    bwa mem -t 32 unicycler.fasta reads_qc/illumina_1.fastq.gz reads_qc/illumina_2.fastq.gz | samtools sort > alignments.bam
    samtools index alignments.bam
    java -Xmx16G -jar pilon-1.24.jar --genome unicycler.fasta --frags alignments.bam --output unicycler_pilon
    seqtk seq -U unicycler_pilon.fasta > temp.fasta && mv temp.fasta unicycler_pilon.fasta
    rm *.bam *.bam.bai *.bwt *.pac *.ann *.amb *.sa

    # POLCA
    polca.sh -a unicycler.fasta -r reads_qc/illumina_1.fastq.gz" "reads_qc/illumina_2.fastq.gz -t 32 -m 1G
    mv *.PolcaCorrected.fa unicycler_polca.fasta
    seqtk seq -U unicycler_polca.fasta > temp.fasta && mv temp.fasta unicycler_polca.fasta
    rm *.fai *.err *.bwa.* *.sam *.success *.bam *.bam.bai *.batches *.names *.report *.vcf
done

# Assessment
for g in ATCC_10708_Salmonella_enterica ATCC_17802_Vibrio_parahaemolyticus ATCC_25922_Escherichia_coli ATCC_33560_Campylobacter_jejuni ATCC_BAA-679_Listeria_monocytogenes; do
    cd ~/ATCC_assemblies/"$g"
    cat "$g".fasta | awk '{if ($0 ~ /^>/) print $0; else print $0$0}' > doubled.fasta
    echo "$g"
    minimap2 -c --eqx -N 100 doubled.fasta unicycler.fasta  2> /dev/null | awk '$8 < $7/2' > unicycler.paf
    printf none"\t"
    count_errors.py unicycler.paf
    minimap2 -c --eqx -N 100 doubled.fasta unicycler_pilon.fasta  2> /dev/null | awk '$8 < $7/2' > unicycler_pilon.paf
    printf pilon"\t"
    count_errors.py unicycler_pilon.paf
    minimap2 -c --eqx -N 100 doubled.fasta unicycler_polca.fasta  2> /dev/null | awk '$8 < $7/2' > unicycler_polca.paf
    printf polca"\t"
    count_errors.py unicycler_polca.paf
    minimap2 -c --eqx -N 100 doubled.fasta unicycler_polypolish.fasta  2> /dev/null | awk '$8 < $7/2' > unicycler_polypolish.paf
    printf polypolish"\t"
    count_errors.py unicycler_polypolish.paf
    printf "\n"
    rm doubled.fasta
done
{% endhighlight %}

</details>


## Results

**ATCC-10708 _Salmonella enterica_**

| Polisher          | Errors | Accuracy | Result            |
|-------------------|--------|----------|-------------------|
| none[^salmonella] | 135    | Q45.45   |                   |
| Pilon             | 135    | Q45.45   | same              |
| POLCA             | 143    | Q45.20   | worse (+8 errors) |
| Polypolish        | 135    | Q45.45   | same              |


**ATCC-17802 _Vibrio parahaemolyticus_**

| Polisher      | Errors | Accuracy | Result             |
|---------------|--------|----------|--------------------|
| none[^vibrio] | 341    |   Q41.72 |                    |
| Pilon         | 348    |   Q41.63 | worse (+7 errors)  |
| POLCA         | 366    |   Q41.41 | worse (+25 errors) |
| Polypolish    | 342    |   Q41.71 | worse (+1 error)   |


**ATCC-25922 _Escherichia coli_**

| Polisher     | Errors | Accuracy | Result             |
|--------------|--------|----------|--------------------|
| none[^ecoli] | 1      |   Q67.11 |                    |
| Pilon        | 22     |   Q53.68 | worse (+21 errors) |
| POLCA        | 25     |   Q53.13 | worse (+24 errors) |
| Polypolish   | 5      |   Q60.12 | worse (+4 errors)  |


**ATCC-33560 _Campylobacter jejuni_**

| Polisher     | Errors | Accuracy | Result            |
|--------------|--------|----------|-------------------|
| none[^campy] | 1      |   Q62.38 |                   |
| Pilon        | 1      |   Q62.38 | same              |
| POLCA        | 2      |   Q59.37 | worse (+1 error)  |
| Polypolish   | 1      |   Q62.38 | same              |


**ATCC-BAA-679 _Listeria monocytogenes_**

| Polisher   | Errors | Accuracy | Result            |
|------------|--------|----------|-------------------|
| none       | 0      |       Q∞ |                   |
| Pilon      | 0      |       Q∞ | same              |
| POLCA      | 0      |       Q∞ | same              |
| Polypolish | 0      |       Q∞ | same              |



## Discussion and conclusions

It’s important to note that short-read contigs can contain errors, often due to collapsed repeat sequences and incorrect tandem repeat counts. When I developed Unicycler, I assumed that SPAdes contigs would be reliable, i.e. repeats and ambiguity would cause assembly fragmentation but not errors _in_ contigs. So a Unicycler hybrid assembly doesn't change the SPAdes contigs, it just scaffolds them together with long reads. But it's now clear to me that errors can and do occur in SPAdes contigs, so those errors will also end up in a Unicycler hybrid assembly. That's why I now recommend a long-read-first approach to hybrid assembly.[^hybrid]

As for the effect of short-read polishing on short-read assemblies, the results were clear: polishing either made no difference or it introduced errors. There was not a single case where polishing reduced the error count. Polypolish introduced the fewest total errors (5) and POLCA introduced the most (58). Introduced errors were mostly found in or near repetitive sequences, once again highlighting that repeats are the bane of genome assembly.[^repeats] So my recommendation is simple: don't polish after a short-read Unicycler assembly!




## Footnotes

[^depth]: 130× to 700× for ONT reads, 550× to 820× for Illumina reads.

[^double]: I actually use a doubled version of the reference genome, where each circular sequence is duplicated. This allows for a contig which spans the start/end boundary to align in a single alignment. We did something similar in our [long-read assembler benchmarking paper](https://f1000research.com/articles/8-2138/v4) – see Figure S6.

[^count]: I counted unaligned parts plus all non-match CIGAR parts (`X`, `I` and `D`) as errors. I also counted each base of an insertion/deletion as an error, e.g. a 100-bp deletion counts as 100 errors. This makes my error counts and corresponding accuracy qscores very unforgiving in the case of structural errors.

[^salmonella]: All 135 pre-polishing errors in the _Salmonella_ assembly were in the plasmid contig, which failed to circularise in the Illumina-only assembly and also contained some errors. These seem to be related to very low Illumina read depth in the affected regions.

[^vibrio]: The 341 pre-polishing errors in the _Vibrio_ assembly occurred at 3 locations. The first was an `AACAGC` tandem repeat – based on long reads, there are 32 copies, but the Illumina assembly cut this down to 21 copies. The second was an inexact 134-bp tandem repeat – based on long reads, there are ~6.5 copies, but the Illumina assembly cut this down to 4.5 copies. The third was a cluster of 7 mismatches. I can't explain this one, other than to say the Illumina reads in this region have a lot of errors. Curiously, the low-<em>k</em>-mer SPAdes graph (_k_=27) got the sequence right, but the final graph (_k_=127) got it wrong.

[^ecoli]: The one pre-polishing error in the _E. coli_ assembly was in the rRNA operon, a 7× repeat. This was collapsed down to a single contig in the Illumina-only assembly, but there is some variation in the operon, so the Illumina contig represents a consensus sequence, and that consensus is not an exact match for any of the actual instances.

[^campy]: The one pre-polishing error in the _Campylobacter_ assembly was in a G×10 homopolymer: my 'truth' assembly had G×10 and the Illumina-only assembly had G×11. The Illumina reads were pretty evenly split between G×10 and G×11. I decided that G×10 was better because the R10.4.1 reads supported that length and the corresponding open reading frame was longer with G×10. But there may be heterogeneity here, i.e. both lengths are correct.

[^hybrid]: For details on long-read-first hybrid assembly, see our 'perfect assembly' [paper](https://doi.org/10.1371/journal.pcbi.1010905) and accompanying [online tutorial](https://github.com/rrwick/Perfect-bacterial-genome-tutorial/wiki).

[^repeats]: This is why long reads are so much better for assembly: they can usually span all repeats in a bacterial genome, which effectively means there are no repeats.

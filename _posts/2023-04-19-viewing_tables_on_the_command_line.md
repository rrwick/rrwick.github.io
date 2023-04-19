---
layout: post
title:  "Viewing tables on the command line"
date:   2023-04-19
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7844352.svg)](https://doi.org/10.5281/zenodo.7844352)



In bioinformatics, I often find myself working with tab-delimited text files: [SAM alignments](https://samtools.github.io/hts-specs/SAMv1.pdf), [VCF variant calls](https://samtools.github.io/hts-specs/VCFv4.2.pdf), [minimap2](https://github.com/lh3/minimap2)'s [PAF alignments](https://github.com/lh3/miniasm/blob/master/PAF.md), ONT Guppy's `sequencing_summary.txt` files, Verticall's big [pairwise TSV files](https://github.com/rrwick/Verticall/wiki/Columns-in-pairwise-tsv-file), [Kleborate](https://github.com/klebgenomics/Kleborate)'s output file, and many many more. This post is to share my approach for viewing these files on the command line in a friendly manner.

These are the features I want in a command-line TSV viewer:
* Nicely aligns columns of data.
* Keeps the header visible when I scroll down.
* Handles large files, e.g. 1000000+ rows.
* Handles very wide columns.
* Loads files directly or accepts input from stdin.
* Can handle particular bioinformatics formats (e.g. SAM and VCF).
* Also works on CSV files.
* Lets you use [less](https://www.greenwoodsoftware.com/less) features like searching.

Googling led me to some solutions based on the [column](https://man7.org/linux/man-pages/man1/column.1.html) utility[^1], but I couldn't get all of the above features, so I had to write my own Bash functions.



## Prerequisites

After some experimenting, I landed on an approach that uses [xsv](https://github.com/BurntSushi/xsv) to space the columns[^2] and [less](https://www.greenwoodsoftware.com/less) to view the result. But I needed to build both those tools from source: xsv has [a column alignment bug](https://github.com/BurntSushi/xsv/issues/151) that is fixed by using the latest version of the [csv](https://crates.io/crates/csv) crate, and only recent versions of less have the [header](https://unix.stackexchange.com/a/739599/316196) option.

Assuming you have [Rust](https://www.rust-lang.org/) and command-line development tools (e.g. make and gcc) installed, here is how you can build xsv and less:
```bash
git clone https://github.com/BurntSushi/xsv
cd xsv
sed -i 's/csv = "1"/csv = "1.2"/' Cargo.toml  # ensure latest version of csv crate
cargo build --release
cp target/release/xsv ~/.local/bin  # adjust as necessary based on where you want the executable

wget https://www.greenwoodsoftware.com/less/less-608.tar.gz
tar -xvf less-608.tar.gz
cd less-608
./configure --prefix=~/.local  # adjust as necessary based on where you want the executable
make -j
make install
```

You will also need to be using GNU versions of [sed](https://www.gnu.org/software/sed) and [grep](https://www.gnu.org/software/grep). This is particularly relevant for macOS which ships with BSD versions of these tools. On my Mac, I installed the GNU versions with [Homebrew](https://brew.sh):
```bash
brew install grep
brew install gnu-sed
echo 'export PATH=/opt/homebrew/opt/gnu-sed/libexec/gnubin:"$PATH"' >> ~/.zshrc
echo 'export PATH=/opt/homebrew/opt/grep/libexec/gnubin:"$PATH"' >> ~/.zshrc
```



## Bash functions

I wrote[^3] a function named `tv` (for 'table viewer') and put it in my `.zshrc` file. Here it is, with an abundance of explanatory comments:
```bash
# Table viewer: a Bash function to view TSV/CSV files/data on the command line
tv () {

    # The default behaviour is to treat the first line of the input as a header
    # line. But the header line count can be provided as a number, e.g. `tv 0`
    # for no header or `tv 2` for two header lines.
    header_count=1
    if [[ "$#" -eq 2 ]] || ([[ "$#" -eq 1 ]] && [[ ! -t 0 ]]); then
        header_count="$1"
        shift
    fi

    # If being run on a file (not stdin), make sure the file exists.
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi
    
    # This function can be run on a file or on stdin. For file inputs, the
    # first line is gotten using `head`. For stdin inputs, the first line is
    # gotten using the `read` command. The input type is stored in the
    # `file_input` variable, because this is needed later in this function.
    if [[ -t 0 ]]; then
        file_input=true
        first_line=$(head -n1 "$1")
    else
        file_input=false
        read -r first_line
    fi

    # This function works on both TSV (tab-delimited) and CSV (comma-delimited)
    # inputs. If a tab is in the first line, then TSV is assumed. If the first
    # line has no tab but does have a comma, then CSV is assumed. If neither
    # are present, it defaults to TSV (i.e. just one column).
    delimiter="\t"
    if [[ ! $first_line == *$'\t'* && $first_line == *,* ]]; then
        delimiter=","
    fi

    # The `xsv table` command is used to space the columns nicely, and `less`
    # is used to view the result in an interactive manner. `2>&1` is used with
    # `xsv table` to ensure that error messages (e.g. inconsistent column
    # counts) appear in the interactive view (otherwise they would not be seen
    # until exiting `less`). If the function's input is from stdin, both the 
    # first line and the remainder of the input are passed to `xsv table`. The
    # `-K` flag makes `less` quit immediately when the user presses Ctrl+C.
    if [[ "$file_input" = true ]] ; then
        xsv table -d"$delimiter" "$1" 2>&1 | less -S -K --header "$header_count"
    else
        {
            echo "$first_line"
            cat
        } | xsv table -d"$delimiter" 2>&1 | less -S -K --header "$header_count"
    fi
}
```

And a few more related functions[^4]:
* `pafv` for [PAF files](https://github.com/lh3/miniasm/blob/master/PAF.md)
* `samv` for [SAM files](https://samtools.github.io/hts-specs/SAMv1.pdf)
* `varv` for [VCF files](https://samtools.github.io/hts-specs/VCFv4.2.pdf)
* `body` is a function I got from [Stack Exchange](https://unix.stackexchange.com/a/11859/316196) that lets you pipe data while leaving the first line alone – very useful for leaving the header line in place when sorting or filtering.

```bash
# PAF viewer: a Bash function to view PAF files/data on the command line
pafv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi

    {
        # PAF files don't have a header line, so one is printed for easier
        # viewing.
        printf "QNAME\tQLEN\tQSTART\tQEND\tSTRAND\t"
        printf "TNAME\tTLEN\tTSTART\tTEND\t"
        printf "MATCHES\tALEN\tMAPQ\tTAGS\n"

        # PAF files contain key:value tags at the end of each line, but these
        # can be variable in number. Replacing 13th and later tabs with spaces
        # ensures that the output has exactly 13 columns and is therefore a
        # valid TSV.
        cat "$@" | sed 's/\t/ /g13'
    } | tv
}


# SAM viewer: a Bash function to view SAM/BAM files/data on the command line
samv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi

    {
        # SAM files don't have a header line, so one is printed for easier
        # viewing.
        printf "QNAME\tFLAG\tRNAME\tPOS\tMAPQ\tCIGAR\t"
        printf "RNEXT\tPNEXT\tTLEN\tSEQ\tQUAL\tTAGS\n"

        # SAM files contain meta-information lines at the top (start with `@`)
        # which must be removed to view the file as a TSV. SAM files also
        # contain key:value tags at the end of each line, but these can be
        # variable in number. Replacing 12th and later tabs with spaces ensures
        # that the output has exactly 12 columns and is therefore a valid TSV.
        grep -vP "^@" "$@" | sed 's/\t/ /g12'
    } | tv
}


# VCF viewer: a Bash function to view VCF files/data on the command line
varv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi

    # VCF files contain meta-information lines at the top (start with `##`)
    # which must be removed to view the file as a TSV. Additionally, the
    # leading `#` is removed from the header line.
    grep -vP "^##" "$@" | sed s'/^#//' | tv
}


# Print header and run command on body (unix.stackexchange.com/a/11859/316196)
body () {
    IFS= read -r header
    printf '%s\n' "$header"
    "$@"
}
```



## Usage examples

`tv` can be run directly on TSV and CSV files:
```bash
tv data.tsv
tv data.csv
```

And it can accept data from stdin:
```bash
cat data.tsv | tv
tv < data.tsv
```

Compressed files cannot be viewed directly[^5], so you must use stdin to view compressed files:
```bash
tv data.tsv.gz                # will NOT work
zcat data.tsv.gz | tv         # will work
bunzip2 -c data.tsv.bz2 | tv  # will work
```

Include a number after `tv` to change the header line count (defaults to 1):
```bash
tv 0 data.tsv
tv 2 data.tsv
cat data.tsv | tv 0
cat data.tsv | tv 2
```

Use the `body` function to leave the first line alone when piping:
```bash
cat data.tsv | body sort | tv
cat data.tsv | body awk '$1 > 1000' | tv
```

`pafv`, `samv` and `varv` add some pre-processing specific to bioinformatics formats:
```bash
pafv alignments.paf
cat alignments.paf | awk '$10/$11 > 0.9' | pafv
samv alignments.sam
samtools view alignments.bam | samv
varv variants.vcf
```

Since `tv` uses [less](https://www.greenwoodsoftware.com/less), type `q` to quit the interactive display, and you can use all of less's features, e.g. [searching](https://linuxhandbook.com/search-less-command). These functions are also quick – on my laptop, a 1 GB TSV file only takes a few seconds to load and display.

These functions work for me on macOS and Linux, and while I use Zsh as my main shell, I also tested the functions in Bash. So assuming you meet the prerequisites (e.g. GNU sed and grep, see above), I think they will work most Unix-like systems.

I hope these functions will be useful to others! If you'd like to use them, here they are without the comments for you to put in your shell's configuration file:

<details>
<summary>functions without comments</summary>

{% highlight bash %}
# https://rrwick.github.io/2023/04/19/viewing_tables_on_the_command_line.html
tv () {
    header_count=1
    if [[ "$#" -eq 2 ]] || ([[ "$#" -eq 1 ]] && [[ ! -t 0 ]]); then
        header_count="$1"
        shift
    fi
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi
    if [[ -t 0 ]]; then
        file_input=true
        first_line=$(head -n1 "$1")
    else
        file_input=false
        read -r first_line
    fi
    delimiter="\t"
    if [[ ! $first_line == *$'\t'* && $first_line == *,* ]]; then
        delimiter=","
    fi
    if [[ "$file_input" = true ]] ; then
        xsv table -d"$delimiter" "$1" 2>&1 | less -S -K --header "$header_count"
    else
        {
            echo "$first_line"
            cat
        } | xsv table -d"$delimiter" 2>&1 | less -S -K --header "$header_count"
    fi
}

pafv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi
    {
        printf "QNAME\tQLEN\tQSTART\tQEND\tSTRAND\t"
        printf "TNAME\tTLEN\tTSTART\tTEND\t"
        printf "MATCHES\tALEN\tMAPQ\tTAGS\n"
        cat "$@" | sed 's/\t/ /g13'
    } | tv
}

samv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi
    {
        printf "QNAME\tFLAG\tRNAME\tPOS\tMAPQ\tCIGAR\t"
        printf "RNEXT\tPNEXT\tTLEN\tSEQ\tQUAL\tTAGS\n"
        grep -vP "^@" "$@" | sed 's/\t/ /g12'
    } | tv
}

varv () {
    if [[ -t 0 && ! -e $1 ]]; then
        echo "Error: file '$1' does not exist" >&2
        return 1
    fi
    grep -vP "^##" "$@" | sed s'/^#//' | tv
}

body () {
    IFS= read -r header
    printf '%s\n' "$header"
    "$@"
}
{% endhighlight %}

</details>




## Footnotes

[^1]: [This blog post](https://www.stefaanlippens.net/pretty-csv.html) by Stefaan Lippens describes some functions that use the column utility.

[^2]: As [Antonio Camargo pointed out](https://twitter.com/apcamargo_/status/1648713930930540546), [csvtk](https://github.com/shenwei356/csvtk) can also do this. But I tested the performance of xsv and csvtk on a big file, and xsv was more than 10 times faster.

[^3]: I'm not great with Bash syntax, so I wrote this with some help from GPT-4. As of April 2023, coding with an AI assistant is still pretty new to me, but it was fun! GPT-4 often made mistakes, but it still saved me a lot of Googling and helped me write Bash code faster than I could on my own.

[^4]: I had originally named the functions `pv`, `sv` and `vv` for brevity, but these have some conflicts ([thanks, Bram van Dijk](https://twitter.com/bramvandijk88/status/1648664773184180227)), so I've renamed them to `pafv`, `samv` and `varv`.

[^5]: I tried to incorporate this feature, but every working solution I found had performance costs, e.g. it made the `tv` function slow on really big files. So I'm happy to just rely on stdin for compressed files.

---
layout: post
title:  "Mojo: the future of bioinformatics tools?"
date:   2023-06-02
author: Ryan Wick
---

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.7996752.svg)](https://doi.org/10.5281/zenodo.7996752)



## My dream language

I have developed a number of command-line tools for bioinformatics, mostly in [Python](https://www.python.org) but also in [R](https://www.r-project.org), [C++](https://isocpp.org) and [Rust](https://www.rust-lang.org). Based on this experience, I'd give the following criteria for my hypothetical ideal programming language:
1. __Simple high-level syntax.__ Implementations of my projects often evolve over time, requiring lots of experimentation to get right, so I need to be able to develop quickly.
2. __Low-level code as needed.__ Sometimes performance matters, often for just a small piece of the program (e.g. an inner loop). I want to be able to optimise some functions for speed and memory efficiency while keeping the rest of my code in an easy-to-read high-level format.
3. __Big ecosystem of packages.__ If I need to use an existing algorithm, I don't want to have to code it up myself.
4. __Easy-to-use parallelism.__ Modern CPUs have many cores, so it's great to use them for parallelisable tasks. Even better if the GPU can be used when appropriate.
5. __Compiles into a single binary file.__ Nobody will use my tools if they can't install them! Having pre-compiled executable binaries for common platforms makes distribution so much easier.
6. __Popular__. This makes it easy to troubleshoot problems and more likely that other people can read or modify my code.

Python is my go-to language in bioinformatics because it excels at #1, #3 and #6. However, Python is a slow interpreted language, requiring low-level libraries (e.g. [NumPy](https://numpy.org)) to be fast. Parallelism is possible but awkward. Distribution and deployment is a complicated mess with too many puzzle pieces: pip, pipenv, setup.py, pyproject.toml, setuptools, Poetry, eggs, wheels, virtualenv, homebrew, conda, etc! Even as a regular Python developer, I can't keep up.[^xkcd]

For these reasons, I've been drawn to Rust. It's low-level and fast (#2), compiles to easily distributable files[^binary] (#5) and is becoming very popular[^stackoverflow] (#3 and #6). It also has some other good features like memory safety.[^safety] But it's more laborious to write Rust than Python, and the learning curve is steep. I sometimes find myself trying ideas in Python then rewriting in Rust once my program has come into focus. That's a lot of extra work.

What I really want is Python, but with fixes for #2, #4 and #5. [Cython](https://cython.org) is probably the closest thing to this: it's a superset of Python that allows for low-level code as needed. However, I've found Cython challenging to use and never really wrapped my head around it. I've managed to turn simple one-file Python scripts into distributable binaries using Cython, but I haven't pulled this off for big projects.



## Mojo

About a month ago, [Modular](https://www.modular.com) announced a new programming language: [Mojo](https://www.modular.com/mojo). Check out their [product launch video](https://www.youtube.com/watch?v=-3Kf2ZZU-dg&t=1543s) and [this post](https://www.fast.ai/posts/2023-05-03-mojo-launch.html). While new languages are common, there are some heavyweights behind Mojo: [Chris Lattner](https://www.nondot.org/sabre) is a developer and it's endorsed by [Jeremy Howard](https://www.fast.ai/about.html#jeremy-howard).

Mojo promises to meet all of my criteria. It will be a superset of Python, so I could mostly write in the easy Python syntax that I already know (#1 and #6) but with additional low-level code for performance-critical functions (#2). It will be compatible with existing Python packages (#3). It will be able to use CPU threads, SIMD and GPUs (#4). It will compile down to a single binary (#5).[^mojobinary] And it's initially targeting the AI sector, which is booming right now to say the least (#6). So I'm excited!

This all makes me optimistic that I may someday use Mojo as my go-to language for bioinformatics. However, Mojo is not released yet, so it remains to be seen whether it will come to fruition. Even if all goes to plan, it will probably be years before Mojo is mature enough to confidently use. Regardless, I'm rooting for it to succeed and will be eagerly watching for updates!



## Footnotes

[^batteries]: Python is often described as a [batteries included](https://peps.python.org/pep-0206) language.

[^xkcd]: Due to this profusion of options, local Python environments are often a mess: [xkcd.com/1987](https://xkcd.com/1987)

[^binary]: For my Rust-based tools, I put an executable binary on GitHub for common platforms ([example](https://github.com/rrwick/Core-SNP-filter/releases)), so most users don't need to worry about compilation.

[^stackoverflow]: In the [Stack Overflow developer survey](https://survey.stackoverflow.co/2022), Rust is a respectable #14 for most popular language. More impressive, it's #1 for the most loved and most wanted language.

[^safety]: Rust's safety is a big selling point – it is immune to some types of bugs.

[^mojobinary]: Mojo binaries will be small and quick to start, i.e. not just embedding the Python interpreter into the binary like [PyOxidizer](https://pyoxidizer.readthedocs.io). See [this tweet from Jeremy Howard](https://twitter.com/jeremyphoward/status/1653924500789166080)

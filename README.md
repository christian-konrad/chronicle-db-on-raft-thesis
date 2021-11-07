# Log Replication using Raft Consensus Protocol for ChronicleDB, a High-Throughput Time-Series Database

(HA and Strong Consistency in Title???)

Master Thesis written in Markdown and Latex, compilable using Pandoc

### How to build

- Step 1: Write using Pandoc Markdown and LaTeX chapter-wise by editing and adding files in the source directory
- Step 2: Customize your output by editing the markdown preface
- Step 3: Run `make.sh` from the command line
- Step 4: Your shiny PDF waits for you in the `out` directory

# Prerequisitions

Follow instruction on installing pandoc in this link http://pandoc.org/installing.html. For PDF output, you’ll also need to install a LaTeX distribution on your machine. We recommend to use [MiKTeX](https://miktex.org/download) as the boilerplate has been successfully tested there.

After the installation and for the first time running the boilerplate, MiKTeX may ask you to install all the dependencies which can take several minutes if you haven't run it before.

# Configuration options

All configuration options are handled in `01_preface.md`.

## Document title, author etc.

To set the title, subtitle, authors, institute anmd date of the paper, replace the placeholders with your values. Don't forget to edit `\fancyhead[LE,RO]{<YOUR TITLE HERE> \hfill \thepage}` as it handles the page heads.

## Citation Style

You can replace ieee.csl with the Citation Style Language file of your need. Then replace the reference at `csl: [ieee.csl]`.

## Thesis language

Set your language in `header-includes: [...] \usepackage[<YOUR LANGUAGE>]{babel}`

## Further reading

See pandoc, LaTeX and Tikz documentation for more information.

# Writing chapters

The project is organized into chapters. You can adopt this structure and add more chapters by simply adding files with a prefix for sorting. The chapters will be concatenated alphabetically, so you can continue the numbering or also declare sub chapters as 02_a, 02_b and so on, for example. But you are free to write the whole thing in a single document if you want so.

# Citations

To add citations to your bibliography, simply paste them intro `citations.bib`. The boilerplate uses [BibTeX](http://www.bibtex.org/Using/de/) for the bibliography. Platforms like Google Scholar allows you to simply copy the citation code for BibTeX with a single click.

# Tips on running the build script

On Linux, simply run `./make.sh` . On Windows, it is recommend to use *git bash* to run the commands.
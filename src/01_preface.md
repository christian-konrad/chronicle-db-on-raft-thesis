---
title: Replication in data stream management systems\vspace{2ex}
subtitle: > 
    Master Thesis - Log Replication using Raft Consensus Protocol for ChronicleDB, a High-Throughput Event Store\vspace{1ex}
date: May 15th, 2022\vspace{4ex}
author: 
    - Christian Konrad
institute: Philipps-Universität Marburg
header-includes: |    
    \usepackage[utf8]{inputenc}
    \usepackage[english]{babel}    
    \usepackage{amsmath}
    \usepackage{amsthm}
    \usepackage[backend=bibtex,style=numeric]{biblatex}
    \usepackage{graphicx}    
    \usepackage{fancyhdr}
    \pagestyle{fancy}
    \fancyhead{}
    \fancyfoot{}
    \fancyhead[LE,RO]{\leftmark\hfill \thepage}
    \renewcommand{\headrulewidth}{0pt}    
    \usepackage{tikz}
    \usetikzlibrary{shapes,arrows,positioning,calc,fit,babel}
    \usepackage{adjustbox}
    \usepackage{tikzpeople}
    \usepackage{pifont,mdframed}
    \usepackage{listings}
    \usepackage{lmodern}
    \usepackage{xcolor}
    \usepackage{boxedminipage}
    \usepackage{needspace}
    \usepackage{epigraph}
    \setlength\epigraphwidth{8cm}
    \setlength\epigraphrule{0pt}
    \usepackage[raggedright]{titlesec}
    \usepackage[font={small},justification=centering]{caption}
    \usepackage{glossaries}
    \usepackage{titling}
    \lstset{
      columns=fullflexible,
      frame=single,
      breaklines=true,
      postbreak=\mbox{\textcolor{red}{$\hookrightarrow$}\space},
      numbers=left,
      numberstyle=\tiny\color{gray},
      keywordstyle=\color[RGB]{40,40,255},
      basicstyle=\ttfamily\small,
      numberstyle=\footnotesize\color{darkgray},
      commentstyle=\it\color[RGB]{0,96,96},
      stringstyle=\slshape\color[RGB]{128,0,0},
      showstringspaces=false,
      aboveskip=20pt,
      belowskip=20pt,
      language=java
    }
    \newenvironment{warning}
      {\par\begin{mdframed}[linewidth=1pt,linecolor=red]%
        \begin{list}{}{\leftmargin=1cm
          \labelwidth=\leftmargin}\item[\Large\ding{43}]}
      {\end{list}\end{mdframed}\par}
bibliography: [citations.bib]
csl: [ieee.csl]    
classoption: [symmetric]
documentclass: report
link-citations: true
numbersections: true
secnumdepth: 4
indent: true
hyperrefoptions:
- breaklinks
---

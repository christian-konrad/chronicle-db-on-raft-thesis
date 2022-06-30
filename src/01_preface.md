---
title: Replication in Data Stream Management Systems\vspace{2ex}
subtitle: > 
    Log Replication using Raft Consensus Protocol for ChronicleDB, a High-Throughput Event Store\vspace{1ex}
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
    \usepackage{scalerel}
    \usepackage{tikz}
    \usetikzlibrary{shapes,arrows,positioning,calc,fit,babel}
    \usepackage{adjustbox}
    \usepackage{enumitem}
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
    \usepackage{todonotes}
    \usepackage{booktabs}
    \usepackage{draftwatermark}    	
    \SetWatermarkColor[gray]{0.95}
    \usepackage{lmodern}
    \usepackage{helvet}
    \usepackage[bitstream-charter,sfscaled=false]{mathdesign}
    \usepackage{mathtools}
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
secnumdepth: 3
indent: true
# Avoid widows and orphans (single line on bottom or top of page, respectively) at almost any cost
clubpenalty: 10000
widowpenalty: 10000
hyperrefoptions:
- breaklinks
---
\setlength{\baselineskip}{15pt}
\usetikzlibrary{svg.path}
\definecolor{orcidlogocol}{HTML}{A6CE39}
\tikzset{
  orcidlogo/.pic={
    \fill[orcidlogocol] svg{M256,128c0,70.7-57.3,128-128,128C57.3,256,0,198.7,0,128C0,57.3,57.3,0,128,0C198.7,0,256,57.3,256,128z};
    \fill[white] svg{M86.3,186.2H70.9V79.1h15.4v48.4V186.2z}
                 svg{M108.9,79.1h41.6c39.6,0,57,28.3,57,53.6c0,27.5-21.5,53.6-56.8,53.6h-41.8V79.1z M124.3,172.4h24.5c34.9,0,42.9-26.5,42.9-39.7c0-21.5-13.7-39.7-43.7-39.7h-23.7V172.4z}
                 svg{M88.7,56.8c0,5.5-4.5,10.1-10.1,10.1c-5.6,0-10.1-4.6-10.1-10.1c0-5.6,4.5-10.1,10.1-10.1C84.2,46.7,88.7,51.3,88.7,56.8z};
  }
}

\newcommand\orcid[1]{\href{https://orcid.org/#1}{\mbox{\scalerel*{
\begin{tikzpicture}[yscale=-1,transform shape]
\pic{orcidlogo};
\end{tikzpicture}
}{|}}}}

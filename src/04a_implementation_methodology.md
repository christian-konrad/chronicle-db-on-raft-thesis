## Methodology {#sec:methodology}

\epigraph{To think, you have to write. If you're thinking without writing, you only think you're thinking.}{--- \textup{Leslie Lamport}}

<!-- Restate your thesis or research problem -->
The main goal of this thesis is to identify a consistency model and replication protocol suitable for an event store with a specific set of requirements, namely ChronicleDB. This includes a detailed discussion of the advantages and disadvantages of the models in question. This section outlines the methods used to achieve this goal. A huge portion of this discussion already happened the chapter [@sec:background], such as in the subsection [@sec:cost-of-replication], where we layed out the criteria to consider when deciding for a consistency model.

<!-- Explain the approach you chose; Describe how you collected the data you used -->
\paragraph{The Research Approach.}

Throughout this work, we use an exploratory research design. So far, we have explored the realm of distributed systems and replication. In the remainder of this work, we will conduct a qualitative comparative analysis of replication protocol implementations. We also gather the requirements for a distributed version of ChronicleDB and the tolerable range of trade-offs. Based on this assessment, we design a distributed version of ChronicleDB with replication in order to implement it as a prototype.

We use benchmarking as a quantitative method to analyze our prototype implementation and to either confirm or refute our predicted results. In this benchmarking, we compare our implementation with ChronicleDB in standalone mode running om a single node and compare the results with findings in the literature, with respect to other distributed databases, to determine if the performance trade-off we experienced is within the expected range.

<!-- Evaluate and justify the methodological choices you made
Describe the criteria you used in choosing your approach to your research. List any potential weaknesses in your methodology and present evidence supporting your choice. Include a brief evaluation of other methodology you might have chosen. -->

\paragraph{Significance of our Results.}

The results of our qualitative research are significant since we covered a broad range of relevant literature in our analysis. We analyzed both the history and present of replication in distributed systems for fault-tolerance, dependability, consistency and scalability. We examined not only industry standards, but also not-so-popular approaches that have received less attention in the academic environment. 

Our quantitative results are (a) reliable and (b) valid because (a) our benchmarking setup can be easily replicated (see the Appendix for a Jupyter notebook that can be run against our implementation of ChronicleDB) and (b) our results clearly depict the metrics needed to measure trade-offs and overall performance. We also point out to the limitations of our evaluation and possible future steps.

<!-- Discuss any obstacles and their solutions
Outline any obstacles you encountered in your research and list how you overcame them. The problem-solving skills you present in this section strengthen the validity of your research with readers. -->

<!--

\paragraph{Limitations.}

Unfortunately, we didn't managed to make our prototype run in a distributed setup on one of the big cloud vendors. This means that we have run our prototype on virtual nodes on a single machine, which may have relatively distorted the results of our benchmarking due to limited I/O on a single physical machine.
-->

<!-- Cite all sources you used to determine your choice of methodology
The final section of your methodology references the sources you used when determining your overall methodology. This reinforces the validity of your research. -->

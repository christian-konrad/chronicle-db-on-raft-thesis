# Fundamentals {#sec:background}

<!--
Fundamentals / environment and related work: 1/3
• describe methods and techniques that build the basis of your work 
• review related work(!)
-->

<!-- 
Example math (placeholder)

$$ G_{neo} = G_{neo_0} \overset{q_1}{\rightarrow} ... \overset{q_n}{\rightarrow} G_{neo_n} = G_{neo'} $$

Example image

\begin{figure}[h]
  \begin{adjustbox}{width=\textwidth}
      \input{images/example_image.tikz}
  \end{adjustbox}
  \caption{Example image}
  \label{fig:example-image}
\end{figure}

Example listing

\lstinputlisting[label={lst:example-code}, caption={Example code}, captionpos=b]{code_listings/example-code.groovy}
-->

This chapter introduces the reader into the fundamentals of replication and event stores. The chapter itself is divided into 4 sections: [Section 1.1](#sec:why-replication) explores the realm of distributed systems in general and replication in detail, comparing the various protocols, algorithms, and concepts to find a protocol that meets the requirements of event stores and in particular those of ChronicleDB. Event stores and time series databases are discussed and compared in ([Section 1.3](#sec:event-stores)). Both sections are followed by a section dealing with either the protocol (Raft, [Section 1.2](#sec:raft)) or event store (ChronicleDB, [Section 1.4](#sec:chronicle-db)) of choice for this work in detail. The chapter concludes with an overview and provides the foundation for a deep understanding of the requirements, characteristics, trade-offs, and specifics of the protocols so that the reader can follow the decisions in the subsequent chapters to design and implement such a replicated system in an educated manner.

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

This chapter introduces the reader into the basics of replication and event stores. It explores the topic in general, comparing the various protocols, algorithms, and concepts to find a protocol that meets the requirements of the event store that is the subject of this thesis (ChronicleDB). Each of the two topics, replication and event stores, is followed by a subchapter dealing with either the protocol (Raft) or event store of choice for this work. The chapter concludes with an overview for the reader and provides the foundation for a deep understanding of the requirements, characteristics, trade-offs, and specifics of the protocols so that the reader can understand the decisions in the following chapters to design and implement such a replicated system in an informed manner.

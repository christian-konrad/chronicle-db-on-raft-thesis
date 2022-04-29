# Fundamentals {#sec:background}

<!--
Fundamentals / environment and related work: 1/3
• describe methods and techniques that build the basis of your work 
• review related work(!)
-->

- Describe in short requirements to modern software, platforms and databases
  - Served from the cloud
  - Must be high available, fast (independent of geographic region)
  - Must be horizontal scalable (explain the term here)
    - Multiple loads in parallel without throughput detrementation
    - Standalone apps could benefit from multithreaded/concurrency, but only to a certain limit (CPU cores, shared memory etc.)) 
  - Must be fault-tolerant. Consistency is important
    - May reference https://gousios.org/courses/bigdata/dist-databases.html#:~:text=Replication%3A%20Keep%20a%20copy%20of,the%20partitions%20to%20different%20nodes. ?
  - Therefore, in fact, modern applications (served on the web but also enterprise and research software , e.g. in compute clusters) are distributed systems most of the times
- Describe in short todays' challenges of distributed systems
  - TODO find those challenges and differentiate what I've written in the previous paragraph


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

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

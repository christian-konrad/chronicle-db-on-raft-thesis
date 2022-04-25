## Why Replication {#sec:why-replication}

> TODO or call it "Motivation"?

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport} \cite{milojicic2002discussion}}

- Performance
  - response time, throughput 


Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

- Reference https://gousios.org/courses/bigdata/dist-databases.html#:~:text=Replication%3A%20Keep%20a%20copy%20of,the%20partitions%20to%20different%20nodes. (in which subsection?)

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

### Use Cases and Challenges

#### Distributed Systems

A distributed system is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed].

Replication plays an important role in distributed systems. In fact, a distributed system...

- Horizontal Scalability

- Nodes accessible from different geographic regions

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

#### Safety and Reliability

\epigraph{Anything that can go wrong will go wrong.}{--- \textup{Murphy's Law} \cite{milojicic2002discussion}}

High availability requires that your application can handle node failures without interrupting service. Replicating data between nodes ensures that the data remains accessible to a certain extend.

- Fault tolerance
  - Against data corruption
  - Against faulty operations (see byzantine fault in next subchapter)

See [@cristian1991understanding]

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

- Example: Aviation Systems

##### Types of Possible Failures

- Network Partitioning
  - Show examples like in https://kriha.de/docs/seminars/distributedsystemsmaster/reliability/reliability.pdf or in official Raft Interactive Diagram
- OS Crashes
- Application Crashes
- Not finding Consensus
  - Byzantine Fault vs Fail Stop
  - Instruction Failures on CPU, RAM Failures, even Bitwise Failures in Cables (quote the one example here I found recently)

### High Availability and Strong Consistency

- CAP Theorem (TODO reference/citation)
- Results in various consistency models with different correctness properties:
  - **Weak Consistency** - 
  - **Strong Consistency** - 
  - TODO more [@gotsman2016cause]
"requesting stronger consistency in too many places may hurt performance, and requesting it in too few places may violate correctness"

- TODO describe that I opted for strong consistency here
  - in background or later in implementation?
- TODO describe my reasoning for this

"Ideally, we would like replicated databases to provide strong consistency, i.e., to behave as if a single centralised node handles all operations. However, achieving this ideal usually requires synchronisation among replicas, which slows down the database and
even makes it unavailable if network connections between replicas fail [2, 24].
For this reason, modern replicated databases often eschew synchronisation completely; such databases are commonly dubbed eventually consistent [47]. In these databases, a replica performs an operation requested by a client locally without any synchronisation with other replicas and immediately returns to the client; the effect of the operation is propagated to the other replicas only eventually. This may lead to anomalies—behaviours deviating from strong consistency"

### Partitioning and Sharding

- Performance is critical for many applications, especially for distributed ones
- Partitioning and sharding are methods to ensure throughput by providing horizontal scalability
  - Explain horizontal scal.

- Various Partitioning strategies 
  - Reference hashing, Cassandra strategies, time split strategies etc. here
- Strategies can also support geograhical scalability, see following subchapter

TODO diagrams, also my own ones from PPT

### Cross-Cluster Replication

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Overview of Replication Protocols

#### Categorization of Replication Protocols

##### State Machine Replication

##### Consensus Protocols

##### Other Kinds of Protocols

- Active-Passive

#### Concrete Protocols

This section provides the reader an overview of concrete, popular replication protocols. Some of them are used in production for years, others are just subject of academic study.

Section N then will take a closer look on Raft, a consensus-based, state machine replication protocol. The author has chosen to use Raft for replication of the time series database which is the subject of this thesis.

##### Paxos

#### Other Mentionable Protocols

##### Chain Replication

##### Viewstamped Replication

ABC

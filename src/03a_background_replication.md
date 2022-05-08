## Why Replication {#sec:why-replication}

> TODO or call it "Motivation"?

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport} \cite{milojicic2002discussion}}

- Performance
  - response time, throughput 


Maintaining copies of the data on multiple nodes...

- Reference https://gousios.org/courses/bigdata/dist-databases.html#:~:text=Replication%3A%20Keep%20a%20copy%20of,the%20partitions%20to%20different%20nodes. (in which subsection?)

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

### Use Cases and Challenges

#### Distributed Systems

A distributed system is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed].

TODO rephrase

"To achieve availability and horizontal scalability, many modern distributed systems rely on replicated databases, which maintain multiple
replicas of shared data [@]. Depending on the replication protocol, the data is accessible to clients at any of the replicas, and these replicas communicate changes to each other using message passing."

Example use cases for replication in distributed systems are large-scale software-as-a-service applications that use data replicas in geographically distinct locations [@mkandla2021evaluation], applications for mobile devices that keep replicas locally to support fast and offline access, key-value stores that act as a book keeper for shared cluster meta data [@] or social networks where user content is to be distributed to billions of other users, to mention a few.

- Horizontal Scalability

- Nodes accessible from different geographic regions, providing lower latencies to users by serving data from nodes closer to the user...

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

#### Safety and Reliability

\epigraph{Anything that can go wrong will go wrong.}{--- \textup{Murphy's Law} \cite{milojicic2002discussion}}

High availability requires that your application can handle node failures without interrupting service. Replicating data between nodes ensures that the data remains accessible to a certain extend.

- Fault tolerance
  - Against data corruption
  - Against faulty operations (see byzantine fault in next subchapter)

See [@cristian1991understanding]

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

- Example use cases: Aviation Systems, lab measurements with low fault-tolerance...

##### Types of Possible Failures {#sec:possible-failures}

- Network Partitioning
  - Show examples like in https://kriha.de/docs/seminars/distributedsystemsmaster/reliability/reliability.pdf or in official Raft Interactive Diagram
- OS Crashes
- Application Crashes
- Hardware Crashes
- Not finding Consensus
  - Byzantine Fault vs Fail Stop
  - Instruction Failures on CPU, RAM Failures, even Bitwise Failures in Cables (quote the one example here I found recently)

### High Availability and Strong Consistency

TODO rephrase

"Building reliable distributed systems at a worldwide scale demands trade-offs between consistency and availability. Consistency is a property of the distributed system which ensures that every node or replica has the same view of data at a given time, irrespective of which client has updated the data. Trading some consistency for availability can oftentimes lead to dramatic improvements in scalability [@pritchett2008base].

Consistency is an ambiguous term in data systems: in the sense of the ACID model (Atomic, Consistent, Isolated and Durable), it is a very different property than the one described in CAP. In the distributed systems literature in general, consistency is understood as a spectrum of models with different guarantees and correctness properties.

A database consistency model determines the manner and timing in which a successful write or update is reflected in a subsequent read operation of that same value. It describes what values are allowed to be returned by operations accessing the storage, depending on other operations executed previously or concurrently, and the return values of those operations. There exists no one-size-fits-all approach: it is difficult to find one model that satisfies all main challenges associated with data consistency [@mkandla2021evaluation]. 

TODO Two distinct perspectives on consistency models: data-centric vs client-centric... "From the data-centric perspective, the distributed data storage system synchronizes the data access operations from all processes to guarantee correct results. From the client-centric perspective, the system only synchronizes the data access operations of the same process, independently from the other ones, to guarantee their consistency. This perspective is justified because it is often the case that shared updates are rare and access mostly private data." [@campelo2020brief]

Following, a few of those consistency models are mentioned [@steen2007distributed]:

**Weak Consistency** - 
- Data-centric
"As its name indicates, weak consistency offers the lowest possible ordering guarantee, since it allows data to be written across multiple nodes and always returns the version that the system first finds. This means that there is no guarantee that the system will eventually become consistent."

**Eventual Consistency** - [@vogels2009eventually]
- TODO list applications
- Client-centric
"This model states that all updates will propagate through the system and all replicas will gradually become consistent, after all updates have stopped for some time [56, 60]. Although this model does not provide concrete consistency guarantees, it is advocated as a solution for many practical situations [10–12, 24, 60] and has been implemented by several distributed storage systems [21, 28, 35, 43]."

**Causal Consistency** - [@shen2015causal]
- Data-centric
"Causal consistency is the strongest form of consistency that satisfies low Latency, defined as the latency less than the maximum wide-area
delay between replicas [@lloyd2011don]."
"Causal consistency is a model in which a sequential ordering is maintained only between requests that have a causal dependency. Two requests A and B have a causal dependency if at least one of the following two conditions is achieved: (1) both A and B are executed on a single thread and the execution of one precedes the other in time; (2) B reads a value that has been written by A. Moreover, this dependency is transitive, in the sense that, if A and B have a causal dependency, and B and C have a causal dependency, then A and C also have a causal dependency [56, 60]. Thus, in a scenario of an always-available storage system in which requests have causal dependencies, a consistency level stricter than that provided by the causal model cannot be achieved due to trade-offs of the CAP Theorem [5, 48]."


TODO BASE? [@pritchett2008base]

Basically Available, Soft state, Eventually consistent

TODO NoSQL Databases are often BASE and not ACID and therefore not strong consistent...

  - AWS services like S3 where eventual consistent when introduced
  - Maximizing read throughput
  - Returning recent writes is not guaranteed
  - Eventual different responses from different nodes at different points in time

**Strong Consistency** - 
"A strong consistency model like linearizability provides an easy-to-understand guarantee: informally, all
operations behave as if they executed atomically on a
single copy of the data. However, this guarantee comes
at the cost of reduced performance [@attiya1994sequential] and fault tolerance [22] compared to weaker consistency models.  algorithms that
ensure stronger consistency properties among replicas
are more sensitive to message delays and faults in the
network. " 
"To achieve that, an absolute global time order must be maintained" [@lamport1978time]

( TODO like in raft )

Strong consistency is important for online transaction or analytical processing with zero tolerance for invalid states.

- After a succesful write, the written object is immediately accessible from all nodes at the same time. What you write is what you will read.
- TODO more [@gotsman2016cause]
"requesting stronger consistency in too many places may hurt performance, and requesting it in too few places may violate correctness"

  - AWS S3 introduced Strong Consistency in 2021 [@amazon2021s3consistency]
  - Read-after-write and list consistency

- TODO describe that I opted for strong consistency here
  - in background or later in implementation?
- TODO describe my reasoning for this (just reference below explanation of CALM)

"Ideally, we would like replicated databases to provide strong consistency, i.e., to behave as if a single centralised node handles all operations. However, achieving this ideal usually requires synchronisation among replicas, which slows down the database and
even makes it unavailable if network connections between replicas fail"

(The latter does not count for raft if quorum still works)

"For this reason, modern replicated databases often eschew synchronisation completely; such databases are commonly dubbed eventually consistent [47]. In these databases, a replica performs an operation requested by a client locally without any synchronisation with other replicas and immediately returns to the client; the effect of the operation is propagated to the other replicas only eventually. This may lead to anomalies—behaviours deviating from strong consistency"

#### The CAP Theorem

Why can't we always have strong consistency? The CAP theorem (CAP stands for Consistency, Availability and Partition Tolerance), also known as Brewer's Theorem, named after its original author Eric Brewer [@brewer2000towards], asserts that the requirements for strong consistency and high availability cannot be met at the same time and serves as a simplifying explanatory model for consistency decisions that depend on the requirements for the availability of the distributed system in question [@gilbert2002brewer]. 

TODO draw and show the diagram

In general, partition tolerance should always be guaranteed in distributed systems (as described above in the [types of possible failures](#sec:possible-failures)). Therefore, the tradeoff is to be made between consistency and availability. Eric Brewer revisited his thoughts on the theorem, stating that tradeoffs are possible for all three dimensions of the theorem: by explicitly handling partitions, both consistency and availability can be optimized [@brewer2012cap].

The theorem is often interpreted as a proof that eventually consistent databases have better availability properties than strongly consistent databases. There is sound critic that in practice this is only true under certain circumstances: the reasoning for consistency tradeoffs in practical systems should be made more carefully and strong consistency can be reached in more real-world applications then the CAP theorem would allow [@kleppmann2015critique].

#### The CALM Theorem

The CALM theorem (CALM stands for...) is a critic on the CAP theorem that says...

[@hellerstein2019keeping]


### Partitioning and Sharding

- Performance is critical for many applications, especially for distributed ones
- sharding (also known as horizontal partitioning) helps to provide high availability, fault-tolerance, and scalability to large databases in the cloud [bagui2015database]
- Partitioning and sharding are methods to ensure throughput by providing horizontal scalability 
  - Explain horizontal scal.

- Various Partitioning strategies 
  - Reference hashing, Cassandra strategies, time split strategies etc. here
- Strategies can also support geograhical scalability, see following subchapter

TODO obsidian notes here

TODO compare with replication, show later how multi-raft supports both

TODO diagrams, also my own ones from PPT

### Cross-Cluster Replication

Is this geographic replication?

Providing lower latencies to users by serving data from nearby nodes...

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Partial Replication

TODO [@shen2015causal]

### Overview of Replication Protocols

#### Categorization of Replication Protocols

##### State Machine Replication

##### Consensus Protocols

##### Other Kinds of Protocols

- Master-Slave
- Active-Passive (any difference to master-slave?^)

#### Concrete Protocols

This section provides the reader an overview of concrete, popular replication protocols. Some of them are used in production for years, others are just subject of academic study.

Section N then will take a closer look on Raft, a consensus-based, state machine replication protocol. The author has chosen to use Raft for replication of the time series database which is the subject of this thesis.

##### Paxos

#### Other Mentionable Protocols

##### Chain Replication

##### Viewstamped Replication

ABC

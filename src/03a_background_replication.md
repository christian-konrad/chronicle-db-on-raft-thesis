## Replication {#sec:why-replication}

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport} \cite{milojicic2002discussion}}

In this chapter, the ...

Maintaining copies of the data on multiple nodes...

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

### Why Replication?

\todo{Call it "Motivation"?}

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

### Use Cases and Challenges

#### Distributed Systems

A distributed system is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed].

\todo{Rephrase}

"To achieve availability and horizontal scalability, many modern distributed systems rely on replicated databases, which maintain multiple
replicas of shared data [@liu2013replication]. Depending on the replication protocol, the data is accessible to clients at any of the replicas, and these replicas communicate changes to each other using message passing."

To achieve high availability and horizontal scalability, many modern distributed systems rely on replicated databases that maintain multiple
replicas of shared data [@liu2013replication]. Depending on the replication protocol, the data is accessible to clients at each of the replicas, and these replicas communicate changes to each other using message passing.

A distributed database therefore is a distributed system designed to provide read/write access to data.

Example use cases for replication in distributed systems are large-scale software-as-a-service applications with data replicated across data centers in geographically distinct locations [@mkandla2021evaluation], applications for mobile devices that keep replicas locally to support fast and offline access, key-value stores that act as a book keeper for shared cluster meta data [@] or social networks where user content is to be distributed to billions of other users, to mention a few.

- TODO distributed kv store
- TODO social network replication (see fb research paper)


#### Horizontal Scalability

- Nodes accessible from different geographic regions, providing lower latencies to users by serving data from nodes closer to the user...

"C. Replication for Scalability
Replication is not only used to achieve high availability,
but also to make a system more scalable, i.e., to improve
ability of the system to meet increasing performance demands in order to provide acceptable level of response time.
Imagine a situation, when a system operates under so high
workload that goes beyond the system’s capability to handle
it. In such situation, either system performance degrades
significantly or the system becomes unavailable. There are
two general solutions for such scenario: vertical scaling, i.e.,
scaling up, and horizontal scaling, i.e., scaling out.
For vertical scaling, data served on a single server [@liu2013replication]"

replication allows making the service horizontally scalable, i.e., improving service throughput. Apart from this, in some cases, replication allows also to
improve access latency and hence the service quality. When
concerning about service latency, it is always desirable to
place service replicas in close proximity to its clients. This
allows reducing the request round-trip time and hence the
latency experienced by clients

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

Various types of failures with two kinds of faulty processes [@bracha1983resilient]:

**Crash Failure/Fail Stop** 
- a process abruptly stops and does not resume

- OS Crashes
- Application Crashes
- Hardware Crashes

**Byzantine Failure** 
malicious processes can also send false messages
\todo{Rephrase}
The Byzantine Generals Problem is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]
- "A process that experiences a Byzantine failure may send contradictory or conflicting data to other processes. Byzantine failures are far more disruptive."
- Not finding Consensus
  - Byzantine Fault vs Fail Stop
  - Instruction Failures on CPU, RAM Failures, even Bitwise Failures in Cables (quote the one example here I found recently)

There is also the case of **Network Partitioning**
- Show examples like in https://kriha.de/docs/seminars/distributedsystemsmaster/reliability/reliability.pdf or in official Raft Interactive Diagram
\todo{Is network partitioning fail-stop or byzantine? Or something else?}



### High Availability and Consistency

\todo{Rephrase}

"Building reliable distributed systems at a worldwide scale demands trade-offs between consistency and availability. Consistency is a property of the distributed system which ensures that every node or replica has the same view of data at a given time, irrespective of which client has updated the data. Trading some consistency for availability can oftentimes lead to dramatic improvements in scalability [@pritchett2008base].

Consistency is an ambiguous term in data systems: in the sense of the ACID model (Atomic, Consistent, Isolated and Durable), it is a very different property than the one described in CAP. In the distributed systems literature in general, consistency is understood as a spectrum of models with different guarantees and correctness properties, as well as various constraints on performance.

A database consistency model determines the manner and timing in which a successful write or update is reflected in a subsequent read operation of that same value. It describes what values are allowed to be returned by operations accessing the storage, depending on other operations executed previously or concurrently, and the return values of those operations. There exists no one-size-fits-all approach: it is difficult to find one model that satisfies all main challenges associated with data consistency [@mkandla2021evaluation]. 

TODO Two distinct perspectives on consistency models: data-centric vs client-centric... "From the data-centric perspective, the distributed data storage system synchronizes the data access operations from all processes to guarantee correct results. From the client-centric perspective, the system only synchronizes the data access operations of the same process, independently from the other ones, to guarantee their consistency. This perspective is justified because it is often the case that shared updates are rare and access mostly private data." [@campelo2020brief]

Following, a few of those consistency models are mentioned, including those relevant for the work of this thesis [@steen2007distributed]:

**Strong Consistency** - 
"A strong consistency model like linearizability provides an easy-to-understand guarantee: informally, all operations behave as if they executed atomically on a
single copy of the data. However, this guarantee comes at the cost of reduced performance [@attiya1994sequential] [@liu2013replication] and fault tolerance compared to weaker consistency models. Algorithms that ensure stronger consistency properties among replicas are more sensitive to message delays and makes the system vulnerable to network partitions. "
"To achieve that, an absolute global time order must be maintained" [@lamport1978time]

\todo{Like in raft}

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
even makes it unavailable if network connections between replicas fail (= network partitions)". This does not apply to all protocols with strong consistency: Raft, the protocol of choice of this work, allows the system to be responsive even when the network is partitioned under certain constraints [TODO raft paper].

"For this reason, modern replicated databases often eschew synchronisation completely; such databases are commonly dubbed eventually consistent [47]. In these databases, a replica performs an operation requested by a client locally without any synchronisation with other replicas and immediately returns to the client; the effect of the operation is propagated to the other replicas only eventually. This may lead to anomalies—behaviours deviating from strong consistency"

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

\todo{BASE? [@pritchett2008base]}

Basically Available, Soft state, Eventually consistent

TODO NoSQL Databases are often BASE and not ACID and therefore not strong consistent...

  - AWS services like S3 where eventual consistent when introduced
  - Maximizing read throughput
  - Returning recent writes is not guaranteed
  - Eventual different responses from different nodes at different points in time

This more relaxing consistency models in general violate crucial correctness properties, compared with strong consistency. A compromise is to allow multiple consistency levels to coexist in the data store, which can be achieved depending on the use case and constraints of the system. An example is to combine both strong and causal consistency for applications with geographically replicated data in distributed data centers [@bravo2021unistore].


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

##### Consensus Protocols {#sec:consensus-protocols}

"A consensus protocol enables a system of n asynchronous processes, some of which are faulty, to reach agreement." [@bracha1983resilient]

- achieve overall system reliability in the presence of a number of faulty processes
- coordinating processes to reach consensus
  - agree on some data value that is needed during computation

TODO difference between decentralized (raft, paxos... (but care that single leader may be a problem)) and centralized consensus (zookeeper?)

Distributed consensus protocols describe a procedure to reach a common agreement among nodes in a distributed or decentralized multi-agent system.

"we apply consensus to ensure reliability and fault tolerance..."

Popular examples for distributed consennsus protocols are [Paxos](#sec:paxos) and [Raft](#sec:raft), the latter being the focus of this work.

Example applications where consensus is needed:
- Clock synchronisation
- Google PageRank
- Smart Power Grid
- Load Balancing
- Key-Value Stores
- and ultimatively Blockchain
- \todo{Find other consensus examples then the wiki ones}
- Distributed systems in general

<!-- (https://en.wikipedia.org/wiki/Consensus_(computer_science)) -->

Different ways to categorize consensus protocols. One is by the size of the value scope:

**Single-Value Consensus Protocols**
- such as Paxos
- nodes agree on a single value

**Multi-Value Consensus Protocols**
- such as Multi-Paxos or Raft
- agree on not just a single value but a series of values over time
- forming a progressively-growing history
- may be achieved naively by running multiple iterations of a single-valued consensus protocol
- but many optimizations and other considerations such as reconfiguration support can make multi-valued consensus protocols more efficient in practice

The other way to categorize those protocols is by fault tolerance (see [Types of Possible Failures](#sec:possible-failures) for reference):

**Crash-Fault Tolerant (BFT) Consensus Protocols**

For Fail-Stop failure model...

"A consensus protocol tolerating those halting failures must satisfy the following properties:

\todo{Re-Research and rephrase}

**Termination**
Eventually, every correct process decides some value.
**Integrity**
If all the correct processes proposed the same value {\displaystyle v}v, then any correct process must decide {\displaystyle v}v.
**Agreement**
Every correct process must agree on the same value."

⎡(n + 1)/2⎤ correct processes are necessary and sufficient to reach agreement [@bracha1983resilient].

"there is no consensus protocol for the fail-stop case that always terminates within a bounded number of steps" [@bracha1983resilient]
\todo{Is this of interest for this work?}

**Byzantine-Fault Tolerant (BFT) Consensus Protocols**

BFT consensus is defined by the following four requirements:

\todo{Rephrase}

**Termination**: Every non-faulty process decides an output.
**Agreement**: Every non-faulty process eventually decides the same output ˆy.
**Validity**: If every process begins with the same input ˆx, then ˆy = xˆ.
**Integrity**: Every non-faulty process’ decision and the consensus value ˆy must have been proposed by some nonfaulty process.

For any consensus protocol to attain these BFT requirements, ⎡(2n + 1)/3⎤ correct processes/nodes are necessary and sufficient to reach agreement, or in other words N ≥ 3f + 1 where f is the number of Byzantine processes and N the number of total processes/nodes needed to tolerate this number of faulty nodes. This fundamental result was first proved by Pease, Lamport et al. [@pease1980faults] and later adapted to the BFT consensus framework [@bracha1983resilient].

\todo{Mention this is not the focus of this work}

Byzantine-Fault tolerant (BFT) consensus protocols are naturally Crash-Fault tolerant (CFT) [@xiao2020survey]

##### State Machine Replication

State machine replication is... [@schneider1990statemachine] and can be achieved through consensus protocols. 

\todo{Rephrase}

Consensus in distributed computing is a more sophisticated
realization of the aforementioned distributed system. In a
typical distributed computing system, one or more clients
issue operation requests to the server consortium, which
provides timely and correct computing service in response
to the requests despite some of servers may fail. Here the
correctness requirement is two-fold: correct execution results
for all requests and correct ordering of them. According to
Alpern and Schneider’s work on liveness definition [31] in
1985, the correctness of consensus can be formulated into
two requirements: **safety** — every server correctly executes the
same sequence of requests, and **liveness** — all requests should
be served.
To fulfill these requirements even in the presence of faulty
servers, server replication schemes especially state machine
replication (SMR) are often heralded as the de facto solution.
SMR, originated from Lamport’s early works on clock synchronization in distributed systems [@lamport1978time], [33], was formally
presented by Schneider [@schneider1990statemachine] in 1990. Setting in the clientserver framework, SMR sets the following requirements:
1) All servers start with the same initial state;
2) Total-order broadcast/atomic broadcast: All servers receive the same sequence of requests as how they were generated from
clients;
3) All servers receiving the same request shall output the
same execution result and end up in the same state. 
[@xiao2020survey]

TODO image from https://arxiv.org/pdf/1904.04098.pdf

<!-- https://en.wikipedia.org/wiki/State_machine_replication -->

##### Other Kinds of Protocols

- Master-Slave
- Active-Passive (any difference to master-slave?^)

#### Concrete Protocols

This section provides an overview of concrete, popular replication protocols to the reader. Some of them are used in production for years, while others are subject of academic studies only.

The [following chapter](#sec:raft) will then spend a closer look on Raft, a consensus-based state machine replication protocol. The author has chosen to use Raft for replication of the event store which is the subject of this thesis.

##### Paxos {#sec:paxos}

- State machine replication (TODO is it ?)
- Strong consistent
- Single-Value, but also multi-value using Multi-Paxos
- Had been state of the art for a long time for strong consistent replication
- Based on distributed consensus
- original by Leslie Lamport [@lamport1998paxos] and further improvements by himself [@lamport2006fast]
- although he tried to make it understandable [@lamport2001paxos], it is shown that people of any kind (students, engineers...) still struggle to understand it in its fullest [@ongaro2013raft] 
- Formally verified using TLA+, as attached to the paper [@lamport2006fast]

#### Other Mentionable Protocols

##### PBFT

##### Chain Replication

[@van2004chain]

##### Viewstamped Replication

ABC

##### Various Blockchain Protocols

PoW, PoS, PoX... all are BFT if conditions apply (for example, no 51% attack) (most important to prevent those failures!)
TODO this is nice to have, remove if not enough time to mention this. If yes, reference (https://arxiv.org/pdf/1810.03357.pdf and https://arxiv.org/pdf/1904.04098.pdf)

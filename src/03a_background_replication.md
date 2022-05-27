## Replication {#sec:why-replication}

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport} \cite{milojicic2002discussion}}

In this chapter, the ...

Maintaining copies of the data on multiple nodes...

The following paragraphs examine and explain why replication is neccessary...

<!-- ### why replication is neccessary -->

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

"One of the potential benefits of distributed systems is their use in
providing highly-available services, that is, services that are likely to
be up and accessible when needed. Availability is essential to many
computer-based services; for example, in airline reservation systems
the failure of a single computer can prevent ticket sales for a
considerable time, causing a loss of revenue and passenger
goodwill.
Availability is achieved through replication. By having more than
one copy of important information, the service continues to be usable
even when some copies are inaccessible, for example, because of a
crash of the computer where a copy was stored. Various replication
algorithms have been proposed to achieve availability. This work tries to find an algorithm that has desirable performance properties for an event store.... "

The following subsections describe replication use cases and the challenges in these particular cases.

\todo{Do we really need an own subsection to explain distributed systems?}

### Distributed Systems

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


### Horizontal Scalability

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

### Safety and Reliability

\epigraph{Anything that can go wrong will go wrong.}{--- \textup{Murphy's Law} \cite{milojicic2002discussion}}

High availability requires that your application can handle node failures without interrupting service. Replicating data between nodes ensures that the data remains accessible to a certain extend.

- Fault tolerance
  - Against data corruption
  - Against faulty operations (see byzantine fault in next subchapter)

See [@cristian1991understanding]

- Example use cases: Aviation Systems, lab measurements with low fault-tolerance...

\todo{Add line wraps after those titles, or use a different approach to enumerate the paragraphs}

#### Types of Possible Faults {#sec:possible-faults}

To describe a fault-tolerant approach that is useful, it must be specified which faults[^1] the system can tolerate. No system can tolerate all kinds of faults (e.g. if all nodes crash permanently), so we need a model that describes the set of faults allowed. There are different types of such faults and two major models to describe them [@bracha1983resilient]:

\paragraph{Crash Failure/Fail-Stop.} Processes with fail-stop behaviour simply "die" on a fault, i.e. they stop participating in the protocol [@schlichting1983fail]. Such a process stops automatically in response to an internal fault even before the effects of this fault become visible. In asynchronous systems, there is no way to distinguish between a dead process and a merely slow process. In such a system, a process may even appear to have failed because the network connection to it is slow or partitioned. However, designing a system as a fail-stop system helps mitigate long-term faulty behavior and can improve the overall fault-tolerance, performance and usability by making a few assumptions [@candea2003crash]: one approach to assuming such a failure is to send and receive heartbeats and conclude that the absence of heartbeats within a certain period of time means that a process has died. False detections of processes that are thought to be dead but are in fact just slow are therefore possible but acceptable as long as they are reasonable in terms of performance. With a growing number of processes involved in a system, such failure detection approaches can become slower and  lead to more false assumptions, so more advanced error detection methods come into play, for example based on gossiping [@renesse1998gossip] or _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable].

"Studies have shown that a main source of downtime in large scale software systems is caused by intermittent or transient bugs" [@candea2003crash]

Examples for faults causing such a fail-stop failure in a distributed system are Operating System (OS) crashes, application crashes, and hardware crashes. 

\paragraph{Byzantine Faults.} Instead of crashing, faulty processes can also send arbitrary or even malicious messages to other processes, containing contradictory or conflicting data. The effect of the fault becomes visible (while being hard to detect) and, if not handled properly, can negatively impact the future behavior of the faulty system, resulting in a _byzantine failure_. Byzantine failures are far more disruptive to a system. Such faults can only be detected under certain circumstances (see later section)...
\todo{Rephrase}
\todo{Illustration for byzantine generals problem}
The _Byzantine Generals Problem_ is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]
- Not finding Consensus
  - Byzantine Fault vs Fail Stop
  - Instruction Failures on CPU, RAM Failures, even Bitwise Failures in Cables (quote the one example here I found recently)

\todo{Find paper with proof, especially for the byzantine case}
A system of $n$ replicas is considered to be fault-tolerant (or _$k$-resilient_) if no more than $k$ replicas become faulty:
- Byzantine failure: $2k + 1$
- Fail-stop failure: $k + 1$

There is also the case of **Network Partitioning**
- network connections between replica nodes fail
- Show examples like in https://kriha.de/docs/seminars/distributedsystemsmaster/reliability/reliability.pdf or in official Raft Interactive Diagram
\todo{Is network partitioning fail-stop or byzantine? Or something else?}
[https://www.usenix.org/system/files/osdi18-alquraan.pdf]

Catastrophic failures manifest easily:
Overall, we found that network-partitioning faults lead to silent catastrophic failures (e.g., data loss, data corruption, data unavailability, and
broken locks), with 21% of the failures leaving the system in a lasting erroneous state that persists even after the partition heals. Oddly, it is easy for these
failures to occur. A majority of the failures required three or fewer frequently used events (e.g., read, and write), 88% of them can be triggered by isolating a single node, and 62% of them were deterministic



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

TODO Causal+ [@lloyd2011don]
"able to leverage the existence of multiple replicas to distribute the load of read requests"

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

Traditional, non-distributed relational database management systems (RDBMS) support both consistency and availability (as there is either one single node available or none), while they do not support partition tolerance, which is mandatory for distributed databases. 
\todo{Source for RDBMS, clustering etc}

In general, partition tolerance should always be guaranteed in distributed systems (as described above in the [types of possible faults](#sec:possible-faults)). Therefore, the tradeoff is to be made between consistency and availability in the case of a network partition. Eric Brewer revisited his thoughts on the theorem, stating that tradeoffs are possible for all three dimensions of the theorem: by explicitly handling partitions, both consistency and availability can be optimized [@brewer2012cap].

The theorem is often interpreted as a proof that eventually consistent databases have better availability properties than strongly consistent databases. There is sound critic that in practice this is only true under certain circumstances: the reasoning for consistency tradeoffs in practical systems should be made more carefully and strong consistency can be reached in more real-world applications then the CAP theorem would allow [@kleppmann2015critique].

#### The CALM Theorem

The CALM theorem (CALM stands for...) is a critic on the CAP theorem that says...

[@hellerstein2019keeping]


### Partitioning and Sharding

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

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

### Performance-Tradeoff Mitigation

#### Partial Replication

TODO [@shen2015causal]

### Replication Protocols

The next subsections describe the different categories of replication protocols and follow with a discussion of specific protocols.

#### Consensus Protocols {#sec:consensus-protocols}

\todo{Rephrase}

\todo{Talk about processes or nodes?}

<!--
TODO The first studies on consensus protocols happened in the field of distributed, asynchronous systems of processes, but is also applicable to large-scale distributed systems of today
-->

"A consensus protocol enables a system of n asynchronous processes, some of which are faulty, to reach agreement." [@bracha1983resilient]

"Each process starts with some initial value. At the conclusion of the protocol all the working nodes must agree on the same value. "

- achieve overall system reliability in the presence of a number of faulty processes
- coordinating processes to reach consensus
  - agree on some data value that is needed during computation

\todo{Rephrase}

Certain different variations of how to understand a consensus protocol appear in the literature. They differ in the assumed properties of the messaging system, in the type of errors allowed to the processes, and in the notion of what a solution is. 
Fischer et al. investigated deterministic protocols that always terminate within a finite number of steps and showed that "every protocol for this problem has the possibility of nontermination, even with only one faulty process [@fischer1985impossibility]." This is based on the failure detection problem discussed earlier in [types of possible faults](#sec:possible-faults): in such a system, a crashed process cannot be distinguished from a very slow one.

Bracha et al. consider protocols that may never terminate, "but this would occur with probability 0, and the expected termination time is finite. Therefore, they terminate within finite time with probability 1 under certain assumptions on the behavior of the system" [@bracha1983resilient]. This can be achieved by adding characteristics of randomness to the protocol, such as random retry timeouts for failed messages, or by the application of _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable].

\todo{Reference the random timeout thing later in raft, which is one basic characteristic}

\todo{Relation to Atomic Broadcasts (Equivalency)?}

<!--
in systems with crash failures, atomic broadcast and consensus are equivalent problems. 
A value can be proposed by a process for consensus by atomically broadcasting it, and a process can decide a value by selecting the value of the first message which it atomically receives. Thus, consensus can be reduced to atomic broadcast. [@chandra1996unreliable]
Conversely, a group of participants can atomically broadcast messages by achieving consensus regarding the first message to be received, followed by achieving consensus on the next message, and so forth until all the messages have been received. Thus, atomic broadcast reduces to consensus. This was demonstrated more formally and in greater detail by Xavier Défago, et al.[https://doi.org/10.1145%2F1041680.1041682]

The Chandra-Toueg algorithm[6] is a consensus-based solution to atomic broadcast.

The Zookeeper Atomic Broadcast (ZAB) protocol is the basic building block for Apache ZooKeeper, a fault-tolerant distributed coordination service which underpins Hadoop and many other important distributed systems.[8][9]

-->

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

The other way to categorize those protocols is by fault tolerance (see [Types of Possible Faults](#sec:possible-faults) for reference):

**Crash-Fault Tolerant (BFT) Consensus Protocols**

For Fail-Stop failure model...

"A consensus protocol tolerating those halting failures must satisfy the following properties:

\todo{Re-Research and rephrase}

**Termination**
Eventually, every correct process decides some value.
**Integrity**
If all the correct processes proposed the same value $y$, then any correct process must decide $y$.
**Agreement**
Every correct process must agree on the same value."

For a faulty system to still be able to reach consensus, even more nodes are required than initially for a system to be [fault-tolerant](#sec:possible-faults):

$\left \lceil (n + 1)/2 \right \rceil$ correct processes are necessary and sufficient to reach agreement [@bracha1983resilient].

"there is no consensus protocol for the fail-stop case that always terminates within a bounded number of steps" [@bracha1983resilient]
\todo{Is this of interest for this work?}

**Byzantine-Fault Tolerant (BFT) Consensus Protocols**

BFT consensus is defined by the following four requirements:

\todo{Rephrase}

**Termination**: Every non-faulty process decides an output.
**Agreement**: Every non-faulty process eventually decides the same output $y$.
**Validity**: If every process begins with the same input $x$, then $y = x$.
**Integrity**: Every non-faulty process' decision and the consensus value $y$ must have been proposed by some nonfaulty process.

For any consensus protocol to attain these BFT requirements, $\left \lceil (2n + 1)/3 \right \rceil$ correct processes/nodes are necessary and sufficient to reach agreement, or in other words $N \geq 3f + 1$ where $f$ is the number of Byzantine processes and $N$ the number of total processes/nodes needed to tolerate this number of faulty nodes. This fundamental result was first proved by Pease, Lamport et al. [@pease1980faults] and later adapted to the BFT consensus framework [@bracha1983resilient].

\todo{Mention this is not the focus of this work}

Byzantine-Fault tolerant (BFT) consensus protocols are naturally Crash-Fault tolerant (CFT) [@xiao2020survey]


"TODO Use cases: Recent advances in permissioned blockchains [58, 111, 124], firewalls [20, 56], and SCADA systems [9, 92] have shown that Byzantine fault-tolerant (BFT) state-machine replication [106] is not a concept of solely academic interest but a problem of high relevance in practice. [@distler2021byzantine]"

\todo{describe that practical consensus protocols need to incorporate a reconfiguration protocol (and describe what this means)}

#### State Machine Replication

State machine replication is... [@schneider1990statemachine] and can be achieved through consensus protocols. 

\todo{Are all state-machine approaches non-byzantine? I don't think so. What about byzantine raft/paxos?}

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

TODO cite the slides for the State-Machine Approach for fault-tolerance https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

TODO there are both BFT and CFT state machine replication protocols: 
"Recent advances in permissioned blockchains [58, 111, 124], firewalls [20, 56], and SCADA systems [9, 92] have shown that Byzantine fault-tolerant (BFT) state-machine replication [106] is not a concept of solely academic interest but a problem of high relevance in practice. [@distler2021byzantine]"


<!-- https://en.wikipedia.org/wiki/State_machine_replication -->

<!--
#### Other Kinds of Protocols

- Master-Slave
- Active-Passive (any difference to master-slave?^)
- Primary-Copy
- TODO Mention them here or just below?
-->

#### Primary-Copy Replication

The simplest, yet weakest approach. One primary server, N backup (copy) servers.

See section "11.6 Replicated state machines vs. primary copy approach" from https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

TODO incorporate the table State-machine vs. Primary-backup at the end of the slides https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

#### Chain Replication

"Chain replication is a new approach to coordinating
clusters of fail-stop storage servers. The approach is
intended for supporting large-scale storage services
that exhibit high throughput and availability without sacrificing strong consistency guarantees." [@van2004chain]

There are deviations from the original, strong-consistency case, such as ChainReaction, which provides causal+ consistency (see TODO ref to paragraph) and geo-replication, and is able to leverage the presence of multiple replicas to distribute the load of read requests [@almeida2013chainreaction].

"Allows to build a distributed system without external cluster management process"

TODO Comparison with primary-copy and state machine approaches

TODO see https://medium.com/coinmonks/chain-replication-how-to-build-an-effective-kv-storage-part-1-2-b0ce10d5afc3

#### Concrete Protocols

This subsection provides an overview of concrete, popular replication protocols to the reader. Some of them are used in production for years, while others are subject of academic studies only.

The [following chapter](#sec:raft) will then spend a closer look on Raft, a consensus-based state machine replication protocol. The author has chosen to use Raft for replication of the event store which is the subject of this thesis.

##### Paxos {#sec:paxos}

"Paxos is one of the most widely deployed consensus algorithms today (blockchains excluded).
Several Google systems use Paxos, including the Chubby [@burrows2006chubby] lock service and the Megastore [5] and Spanner [20] storage systems"

- State machine replication: Executes the same set of commands in the same order, on multiple participants
- Strong consistent
- Single-Value, but also multi-value using Multi-Paxos (which is therefore the most commonly used version of Paxos)
- Had been state of the art for a long time for strong consistent replication
- Based on distributed consensus
- original by Leslie Lamport [@lamport1998paxos] and further improvements by himself [@lamport2006fast]
- although he tried to make it understandable [@lamport2001paxos], it is shown that people of any kind (students, engineers...) still struggle to understand it in its fullest [@ongaro2013raft] 
- Formally verified using TLA+, as attached to the paper [@lamport2006fast], and also by thirds for Multi-Paxos [@chand2016formal]
- handles fail-stop but not Byzantine failures
- does it has a reconfiguraion protocol?

##### Viewstamped Replication

- State machine replication
- offers a reconfiguration protocol
- handles fail-stop but not Byzantine failures

ABC [@oki1988viewstamped], [@liskov2012viewstamped]

##### Practical Byzantine Fault Tolerance (PBFT)

[@castro1999practical]

##### Blockchain Consensus Protocols

Blockchains are distributed systems par excellence... Using strong consistent consensus protocols to ensure every node reads the same... Handling not only fail-stop, but especially byzantine faults is crucial to those consensus protocols to secure a blockchain.


PoW (like the Nakamoto Consensus Algorithm that powers Bitcoin [@nakamoto2008bitcoin]), PoS, PoX, PBFT (see previous paragraph and [@li2020scalable])... all are BFT if conditions apply (especially, no 51% attack - remember, you need $2k + 1$ nodes to tolerate $k$ byzantine faulty nodes) (most important to prevent those failures!), BFT State Machine Replication protocols..
TODO this is nice to have, remove if not enough time to mention this. If yes, reference (https://arxiv.org/pdf/1810.03357.pdf and https://arxiv.org/pdf/1904.04098.pdf)



[^1]: A _fault_ is the initial root cause, including machine and network problems and software bugs. A _failure_ is the loss of a system service due to a fault that is not properly handled [@farooq2012metrics].

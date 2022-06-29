## Replication {#sec:why-replication}

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport}}
<!-- [@milojicic2002discussion] -->

<!-- ### why replication is neccessary, and what it is-->

This section provides the reader with an overview of the topic of distributed systems, defines the term _replication_, explains why replication is crucial for many of those systems, discusses various replication protocols, and concludes with a literature review of recent research in this area. The goal of this section is to provide the reader with a thorough understanding of what replication is, why, where, and when it is needed, and how replication protocols work.

\todo{Glossary?}

A _distributed system_ is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed]. To achieve this behavior, such a system needs to offer certain levels of _availability_, _consistency_, _scalability_ and _fault-tolerance_; therefore, many modern distributed systems rely heavily on replicated data stores that maintain multiple replicas of shared data [@liu2013replication]. Depending on the replication protocol, the data is accessible to clients at any of the replicas, and these replicas communicate changes to each other using message passing.

Thus, a distributed database is a distributed system that provides read and write access to data. Replication in these databases takes place to varying degrees: from simple replication of metadata and current system state to full replication of all payload data, depending on the requirements and characteristics of the system.

Many modern applications, especially those served from the cloud, are inherently distributed. With edge computing, such applications can be distributed not only to nodes in one or more clusters serving many clients, but also to all of those clients participating in the edge-cloud network. Users of cloud applications expect the same experience independent of their geographic region and also regardless of the amount of data they produce and consume.

In complex distributed systems, there are multiple interoperating replicated _(micro)services_ with different availability and consistency characteristics. Ensuring that these systems and services can reliably work together is the topic of container orchestration (e.g., through Kubernetes [@kubernetes2022github]) and service composition [@nikoo2020survey; @camunda2022zeebe]—which need to be replicated, too—which we will not discuss in this work.

Example use cases for replication in distributed systems (additionally to those mentioned in the [Introduction](#sec:introduction)) include large-scale software-as-a-service applications with data replicated across data centers in geographically distinct locations [@mkandla2021evaluation], applications for mobile clients that keep replicas near the user's location to support fast and efficient access [@silva2022geo], key-value stores that act as bookkeepers for shared cluster metadata [@etcd2022], highly available domain name systems (DNS) [@bi2020craft], blockchains and all their various use cases (see [Blockchain Consensus Protocols](#sec:blockchain-consensus)), or social networks where user content is to be distributed cost-effectively to millions of other users [@khalajzadeh2017cost], to name a few.

The following subsections describe replication use cases and the challenges in these particular cases, before outlining relevant replication protocols.

### Horizontal Scalability

\todo{Verify phrasing of this subsection}

Standalone applications can benefit from _scaling up_ the hardware and leveraging concurrency techniques/multithreading behavior (in this case we speak of _vertical scalability_), but only to a certain physical hardware limit (such as CPU cores, network cards or shared memory). Approaching this limit, these applications and services can no longer be scaled economically or at all, as the hardware costs increase dramatically. In addition, applications running on only one single node pose the risk of a single point of failure, which we want to counter with replication.

To scale beyond this limit, a system must be designed to be _horizontally scalable_, that is, to be distributable across multiple computing nodes/servers (also known as to _scale out_). Ideally, the amount of nodes scales with the amount of users, but to achieve this, certain decisions must be made regarding [consistency models](#sec:consistency) and [partitioning](#sec:partitioning). Legacy vertically scalable applications cannot be made horizontally scalable without rethinking the system design, which includes replication and message passing protocols between the nodes to which the application data is distributed.

With vertical scaling, when you host a data store on a single server and it becomes too large to be handled efficiently, you identify the hardware bottlenecks and upgrade. With horizontal scaling instead, a new node is added and the data is partitioned and split between the old and new nodes.

\todo{Illustration of vert. vs horz. scalaing}

With an increasing number of users served, the number of nodes in a horizontally scalable system grows. In the ideal case, the number of nodes required to keep the perceived performance for users constant increases linearly with the number of distinct data entities served (such as tables or data streams with distinct schemas) [@williams2004web]; in other words, the transaction rate grows linearly with the computational power of the cluster. This is not neccessarily the case when a increasing number of users access the same stream/table or inter-node/inter-partition transactions are executed. In the latter case, vertical scaling or advanced partitioning techniques [@aaqib2019efficient] still come handy. In practise, many applications achieve near-linear scalability.

\todo{NTH: show chart and math for near-linear scalability if time}

\todo{Chart: vertical vs horicontal scalability (i.e. the zeebe one)}

Chart from https://www.liquidweb.com/blog/horizontal-vs-vertical-scaling/

Opting in for horizontal scaling is beneficial for large-scale cloud and edge applications as it is easier to add new nodes to a system instead of scaling up existing ones, even if the initial costs are higher as the infrastructure and algorithms must be implemented in first place.

The system design that enables horizontal scalability is also a key requirement for replication. To achieve fault tolerance and availability through replication, it is required to store replicas of a dataset on multiple nodes, therefore the book keeping and message passing infrastructure that is neccessary for partitioning is also a basic requirement for replication protocols [@liu2013replication].

### Safety and Reliability

\epigraph{Anything that can go wrong will go wrong.}{--- \textup{Murphy's Law}}

Modern real-time distributed systems are expected to be reliable, i.e., to function as expected without interruption, and to be safe, i.e., not to cause catastrophic accidents even if a subsystem misbehaves. For a distributed system to be both reliable and secure, it must be designed to be _fault-tolerant_, i.e., the application must be able to cope with node failures without interrupting service. In addition, it must also be able to withstand faults without operating incorrectly, i.e., responding to user requests with erroneous or malicious content. In large distributed systems, faults will happen - they are unevitable. There is no system that is 100% fault-tolerant. A system that can tolerate at least $k$ faulty nodes is called $k$-_fault-tolerant_.

\todo{Cite for unevitable faults}

To understand how a system can be designed to be both reliable and secure, we review the definitions of reliability and security [@mulazzani1985reliability; @farooq2012metrics]:

\todo{If sufficient time, use actual definition formatting for things like this instead of just paragraphs}

\paragraph{Reliability.} The _reliability_ of a system is the probability that its functions will execute successfully under a given set of environmental conditions (operational profile[^operational-profile]) and over a given time period $t$, denoted by $R(t)$. Here $R(t) = P(T > t), t \geq 0$ where $T$ is a random variable denoting the time to failure. This function is also known as the _survival function_.

<!-- TODO May show a typical survival function as in https://en.wikipedia.org/wiki/Survival_function -->

[^operational-profile]: TODO lorem ipsum...

There are also other dimensions of interest to express the reliability of a system:

\begin{enumerate}[label=(\arabic*)]
  \item The probability of failure-free operation over a specified time interval, as in above definition.
  \item The expected duration of failure-free operation.
  \item The expected number of failures per time interval (failure intensity).
\end{enumerate}

The second can be measured using the _Mean Time Between Failures_ (MTBF). The MTBF is a common and important measure and is especially critical for consensus protocols, a particular class of replication protocols, since the MTBF of a single node must be estimated in advance when defining the protocol's timing criteria,  which will be shown later in [Section 1.2](#sec:raft) when we describe the replication protocol of choice of this work. The MTBF itself is a combined metric: it is the sum of the _Mean Time To Failure_ (MTTF) and the _Mean Time To Repair_ (MTTR).

The MTTF describes the average time a system is operational, therefore the average time between a succesful system start and a failure:

$$ \textrm{MTTF} = \sum \frac{t_{\textrm{down}} - t_{\textrm{up}}}{n} $$ 

where $t_{\textrm{down}}$ is the start of the downtime after a failure, $t_{\textrm{up}}$ the start of the uptime before that failure and $n$ the number of failures.

This only describes the runtime: the repair time, including failure detection, reboot, error fixing and network reconfigurations are described by the MTTR. In some literature, the average amount of time it takes to detect a failure is additionally extracted as a dedicated metric, the _Mean Time To Detect_ (MTTD).

$$ \textrm{MTBF} = \textrm{MTTF} + \textrm{MTTR} + \textrm{MTTD} $$

<!-- TODO Draw relation between the metrics this https://www.researchgate.net/profile/Orogun-Adebola/publication/340902878/figure/fig1/AS:883945164009472@1587760358165/Relationships-between-MTBF-MTTF-and-MTTR.jpg or like this https://i0.wp.com/www.researchgate.net/profile/Seongwoo_Woo4/publication/312022469/figure/fig27/AS:445885806059521@1483318868156/A-schematic-diagram-of-MTTF-MTTR-and-MTBF.png?w=584&ssl=1 -->

Actually, the MTBF can also be expressed as an integral over the reliability function $R(t)$ [@birolini2013reliability], which illustrates the relation between the two metrics:

$$ \textrm{MTBF} = \int_{0}^{\infty} R(t) dt $$

One of the most important design techniques to achieve reliability, both in hardware and software, is redundancy [@cristian1991understanding]. There are two main patterns to achieve redundancy: retry and replication. Retry is _redundancy in time_, while replication is _redundancy in space_. Using a retry strategy, a _failure detector_ oftentimes is just a timeout. In this casemm the MTTR is the timeout interval plus the retry time. In replication, timeouts are also used to detect dead nodes (assuming a fail-stop failure model, as described in the following paragraphs) so they can be rebooted in the background or the network can be reconfigured (by the network itself or an external book keeper or orchestrator), commonly covered by the rules of the replication protocol and oftentimes without having a perceivable impact on reliability and availability for the user.

\paragraph{Safety.} The safety of a system is the probability that no catastrophic accidents will occur during system operation over a specified period of time. Safety looks at the consequences and possible accidents, and how to deal with it.

A high degree of reliability, while necessary, is not sufficient to ensure safety.

Safety is an important requirement especially for critical infrastructure applications. Safety means that the system behavior is as expected and not faulty or even malicious...

\todo{Use https://people.cs.rutgers.edu/~pxk/rutgers/notes/content/fault-tolerance-slides.pdf}

<!--
TODO naive system availability calc. also cite this from reliable source 
P: probability that one server fails= 1 – P= availability of service.
e.g. P = 5% => service is available 95% of the time.
Pn: probability that n servers fail= 1 – Pn= availability of service.
e.g. P = 5%, n = 3 => service available 99.875% of the time
-->


#### Types of Possible Faults {#sec:possible-faults}

<!--
 Fault in the system
can be categorized based on time as below:
•	 Transient: This type of fault occurs once and disappear
•	 Intermittent: This type of fault occurs many time in an irregular way
•	 Permanent: This is the fault that is permanent and brings
system to halt.
-->

To describe a fault-tolerant approach that is useful, it must be specified which faults[^faults] the system can tolerate. No system can tolerate all kinds of faults (e.g. if all nodes crash permanently), so we need a model that describes the set of faults allowed. There are different types of such faults and two major models to describe them [@bracha1983resilient]:

[^faults]: A _fault_ is the initial root cause, including machine and network problems and software bugs. A _failure_ is the loss of a system service due to a fault that is not properly handled [@farooq2012metrics]. The probability of a fault manifesting itself as a failure is not uniform: only a few faults actually cause a system failure, and the actual system downtime is caused by an even smaller group of faults. When thinking about fault detection, it is therefore important to identify and focus on this small, but significant group of faults.

\paragraph{Crash Failure/Fail-Stop.} Processes with fail-stop behaviour simply "die" on a fault, i.e. they stop participating in the protocol [@schlichting1983fail]. Such a process stops automatically in response to an internal fault even before the effects of this fault become visible. In asynchronous systems, there is no way to distinguish between a dead process and a merely slow process. In such a system, a process may even appear to have failed because the network connection to it is slow or partitioned. However, designing a system as a fail-stop system helps mitigate long-term faulty behavior and can improve the overall fault-tolerance, performance and usability by making a few assumptions [@candea2003crash]: one approach to assuming such a failure is to send and receive heartbeats and conclude that the absence of heartbeats within a certain period of time means that a process has died. False detections of processes that are thought to be dead but are in fact just slow are therefore possible but acceptable as long as they are reasonable in terms of performance. With a growing number of processes involved in a system, such failure detection approaches can become slower and  lead to more false assumptions, so more advanced error detection methods come into play, for example based on gossiping [@renesse1998gossip] or _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable].

"Studies have shown that a main source of downtime in large scale software systems is caused by intermittent or transient bugs" [@candea2003crash]

Examples for faults causing such a fail-stop failure in a distributed system are Operating System (OS) crashes, application crashes, and hardware crashes. 

\paragraph{Byzantine Faults.} Instead of crashing, faulty processes can also send arbitrary or even malicious messages to other processes, containing contradictory or conflicting data. The effect of the fault becomes visible (while being hard to detect) and, if not handled properly, can negatively impact the future behavior of the faulty system, resulting in a _byzantine failure_. Byzantine failures are far more disruptive to a system, as these faults can result in silent data corruption leaving users with possibly incorrect results. Such faults can only be detected under certain circumstances (see later section)...
\todo{Rephrase}
\todo{Illustration for byzantine generals problem}
The _Byzantine Generals Problem_ is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]
- Not finding Consensus
  - Byzantine Fault vs Fail Stop
  - Instruction Failures on CPU, RAM Failures, even Bitwise Failures in Cables (quote the one example here I found recently)

\todo{Cite papers with proof, especially for the byzantine case}

A system of $n$ replicas is considered to be fault-tolerant (or $k$-_resilient_) if no more than $k$ replicas become faulty:
- Byzantine failure: $n = 2k + 1$, as while $k$ nodes generate false replies, $k+1$ nodes will still provide a majority vote
- Fail-stop failure: $n = k + 1$, as $k$ nodes can fail and one will still be working

\todo{What about network partitioning here? May describe it below. If network is partitioned and k nodes fail, at least one partition is dead... etc}

This also shows that all fail-stop problems are in the space of byzantine problems, too.

Note that not all authors agree to the binary model of fail-stop vs. byzantine faults, claiming that the fail-stop model is too simple to be sufficient to model the behavior of many systems, while the byzantine model is too generalistic, thus too far away from practical application, and so is byzantine fault-tolerance hard to achieve. At least two other fault models are subject of the literature: the _fail-stutter_ fault model [@arpaci2001fail] is an attempt to provide a middle ground model between these two extremes which also allows for _performance faults_, and the _silent-fail-stutter_ fault model tries to extend it furthermore [@kola2005faults]. Despit their usefulness, both models are not quite popular and research on replication protocols does not take those into account, therefore we won't pay too much attention on them in this work.

/todo{This image https://www.researchgate.net/profile/George-Kola/publication/220768708/figure/fig2/AS:667611377459203@1536182364193/Different-fault-models-and-their-relation.png}

There is also the case of **Network Partitioning**, which can lead to byzantine-like behavior once the partitioning is resolved again.
- network connections between replica nodes fail
- Show examples like in https://kriha.de/docs/seminars/distributedsystemsmaster/reliability/reliability.pdf or in official Raft Interactive Diagram
\todo{Is network partitioning fail-stop or byzantine? Or something else?}
[https://www.usenix.org/system/files/osdi18-alquraan.pdf]

Catastrophic failures manifest easily:
Overall, we found that network-partitioning faults lead to silent catastrophic failures (e.g., data loss, data corruption, data unavailability, and
broken locks), with 21% of the failures leaving the system in a lasting erroneous state that persists even after the partition heals. Oddly, it is easy for these
failures to occur. A majority of the failures required three or fewer frequently used events (e.g., read, and write), 88% of them can be triggered by isolating a single node, and 62% of them were deterministic

##### Disaster Recovery

TODO Disaster Recovery and multi-datacenter replication, which is quite different from fault-tolerance

### High Availability and Consistency {#sec:consistency}

\todo{System availability calc}
https://www.fiixsoftware.com/glossary/system-availability/
https://availability.sre.xyz/ 

\todo{Availability vs reliability}

\todo{Rephrase}

High availability is... reliability...

The common way to achieve high availability is through the replication of data in multiple service replicas. High availability comes hand-in-hand with fault-tolerance, as services remain operational in case of failures as clients can be relayed to other working replicas. 

"One of the benefits of distributed systems is their use in
providing highly-available services, that is, services that are likely to
be up and accessible when needed. Availability is essential to many
computer-based services; for example, in airline reservation systems
the failure of a single computer can prevent ticket sales for a
considerable time, causing a loss of revenue and passenger
goodwill.
Availability is achieved through replication[^load-balancing]. By having more than
one copy of important information, the service continues to be usable
even when some copies are inaccessible, for example, because of a
crash of the computer where a copy was stored."

[^load-balancing]: Load balancing is another technique for achieving high availability through service replication. Since this work only deals with data replication, this topic is not covered here (although some session data must be replicated between these services if they are not stateless in the first place).

Building scalable and reliable distributed systems requires a trade-off between consistency and availability. Consistency is a property of the distributed system that ensures that every node or replica has the same view of the data at a given point in time, regardless of which client updated the data. Deciding to trade some consistency for availability can often lead to dramatic improvements in scalability [@pritchett2008base].

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
- Can increase performance dramatically; see for example kubernetes made eventual consistent instead of strong to meet edge-computing requirements [@jeffery2021rearchitecting]
- Not always neccessary; good system design allows even strong consistent services to work at the edge 

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


### Partitioning and Sharding {#sec:partitioning}

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

- Performance is critical for many applications, especially for distributed ones
- sharding (also known as horizontal partitioning) helps to provide high availability, fault-tolerance, and scalability to large databases in the cloud [bagui2015database]
- Partitioning and sharding are methods to ensure throughput by providing horizontal scalability 
  - Explain horizontal scal.

- Various Partitioning strategies 
  - Reference hashing, Consistent hashing (Cassandra strategy), time split strategies etc. here
- Strategies can also support geograhical scalability, see following subchapter

TODO obsidian notes here

TODO compare with replication, show later how multi-raft supports both

TODO diagrams, also my own ones from PPT

### Cross-Cluster Replication

Is this geographic replication?

When concerning about service latency, it is always desirable to place service replicas in close proximity to its clients. This allows reducing the request round-trip time and hence the latency experienced by clients

Providing lower latencies to users by serving data from nearby nodes...

\todo{Rephrase}
The replication performed by modern Internet services spans across several
geographical locations (geo-replication). This allows for increased availability
and low latency, since clients can communicate with the closest geo-graphical
replica. 


### Edge Computing

\todo{Very brief explanation (copy from later sections)}

- Strong concistency can be a problem
- Good system design can help (see next subsection) or using another consistency layer [@jeffery2021rearchitecting]

### Performance-Tradeoff Mitigation

- As mentioned, strong concistency can be a problem for performance as with higher cluster sizes, offering higher availability, write request latency
significantly increases and throughput decreases similarly (TODO refer to evaluation results)
- Many possible solutions:
  - Weaker consistency
  - Partial replication
    - Only metadata (TODO refer to InfluxDB)
    - Other/causal+ (see next paragraph)
  - Sharding + partitioning
    - Leveraging time splits
  - Multi-layer design (full replicated in the cloud, standalone on the edge...)

#### Partial Replication

TODO [@shen2015causal]

###  Error Correcting Codes

TODO only a short excursion

### Replication Protocols

The next subsections describe the different categories of replication protocols and follow with a discussion of specific protocols.

#### Consensus Protocols {#sec:consensus-protocols}

\todo{Rephrase}

\todo{Talk about processes or nodes?}

<!--
TODO The first studies on consensus protocols happened in the field of distributed, asynchronous systems of processes, but is also applicable to large-scale distributed systems of today
-->

"Consensus is a fundamental problem in fault-tolerant systems: how can servers reach agreement
on shared state, even in the face of failures? This problem arises in a wide variety of systems that
need to provide high levels of availability and cannot compromise on consistency; thus, consensus
is used in virtually all consistent large-scale storage systems."

<!--
Raft diss:
Consensus algorithms for practical systems typically have the following properties:
• They ensure safety (never returning an incorrect result) under all non-Byzantine conditions,
including network delays, partitions, and packet loss, duplication, and reordering.
• They are fully functional (available) as long as any majority of the servers are operational
and can communicate with each other and with clients. Thus, a typical cluster of five servers
can tolerate the failure of any two servers. Servers are assumed to fail by stopping; they may
later recover from state on stable storage and rejoin the cluster.
• They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme
message delays can, at worst, cause availability problems. That is, they maintain safety under
an asynchronous model [71], in which messages and processors proceed at arbitrary speeds.
• In the common case, a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls; a minority of slow servers need not
impact overall system performance.
-->

"A consensus protocol enables a system of n asynchronous processes, some of which are faulty, to reach agreement." [@bracha1983resilient]

"Each process starts with some initial value. At the conclusion of the protocol all the working nodes must agree on the same value. "

- achieve overall system reliability in the presence of a number of faulty processes
- coordinating processes to reach consensus
  - agree on some data value that is needed during computation

\todo{Rephrase}

Certain different variations of how to understand a consensus protocol appear in the literature. They differ in the assumed properties of the messaging system, in the type of errors allowed to the processes, and in the notion of what a solution is. 
Fischer et al. investigated deterministic protocols that always terminate within a finite number of steps and showed that "every protocol for this problem has the possibility of nontermination, even with only one faulty process [@fischer1985impossibility]." This is based on the failure detection problem discussed earlier in [types of possible faults](#sec:possible-faults): in such a system, a crashed process cannot be distinguished from a very slow one.

<!-- TODO: bring this and the upper and lower paragraphs into context again
Theorem: In a purely asynchronous distributed system, the consensus problem is impossible to solve if even a single process crashes [@fischer1985impossibility]. -->

Bracha et al. consider protocols that may never terminate, "but this would occur with probability 0, and the expected termination time is finite. Therefore, they terminate within finite time with probability 1 under certain assumptions on the behavior of the system" [@bracha1983resilient]. This can be achieved by adding characteristics of randomness to the protocol, such as random retry timeouts for failed messages, or by the application of _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable].

\todo{Reference the random timeout thing later in raft, which is one basic characteristic}

\todo{Relation to Atomic Broadcasts (Equivalency to Byzanzine Fault Tolerant Consensus)?}

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

\todo{Chart on byzantine quorums from https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L9-bft.pdf}

\todo{Mention this is not the focus of this work}

Byzantine-Fault tolerant (BFT) consensus protocols are naturally Crash-Fault tolerant (CFT) [@xiao2020survey]


"TODO Use cases: Recent advances in permissioned blockchains [58, 111, 124], firewalls [20, 56], and SCADA systems [9, 92] have shown that Byzantine fault-tolerant (BFT) state-machine replication [106] is not a concept of solely academic interest but a problem of high relevance in practice. [@distler2021byzantine]"

\todo{describe that practical consensus protocols need to incorporate a reconfiguration protocol (and describe what this means)}

TODO describe the situation of weighted quorums, for example as it is in Proof-of-Stake

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

#### Read One / Write All (ROWA)

The simplest, yet weakest approach.
"Update anywhere, anytime, anyhow" - Lazy Group ? 
- Simplest approach (simpler than primary copy)
- No availability if one node crashes (cannot write)
  - strong dependency of a write on the availability of all replicas
  - As client writes itself, transaction time increases linearly by number of nodes
- Reading is easy and cheap

#### Primary-Copy Replication

"Update anywhere-anytime-anyway transactional replication has unstable behavior as the workload scales up: a ten-fold increase in nodes and traflc gives a thousand fold increase in deadlocks or reconciliations. Master copy replication (primary copyj schemes reduce this problem." https://dl.acm.org/doi/pdf/10.1145/233269.233330

Another simple, but weak approach. One primary server, N backup (copy) servers.

See section "11.6 Replicated state machines vs. primary copy approach" from https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

TODO incorporate the table State-machine vs. Primary-backup at the end of the slides https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

TODO different use cases then consensus: Not primarily for fault-tolerance, but to provide disaster recovery (having backups) or local copies of the data (to improve performance + availability, e.g. for mobile users)

Primary copy server can easily become a bottleneck in the system. It does not scale with the number of nodes and has a detrimental impact especially if faulty behavior appears. What happens if the primary copy server fails? Failure detection mechanisms are required to detect such situations and subsequently select an alternate primary copy server, creating the risk of multiple primary copies when different network partitions occur; a problem that is mitigated in consensus protocols, as described in the corresponding sections.

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

The [following subchapter](#sec:raft) will then spend a closer look on Raft, a consensus-based state machine replication protocol. The author has chosen to use Raft for replication of the event store which is the subject of this thesis.

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

##### ZooKeeper Atomic Broadcast (ZAB)

TODO describe zookeeper here in short (mostly just point to references), and cite it in the paragraph on Kafka

##### Viewstamped Replication

- Primary-Copy https://pmg.csail.mit.edu/vr/oki88vr-abstract.html
- State machine replication?????
- offers a reconfiguration protocol
- handles fail-stop but not Byzantine failures

ABC [@oki1988viewstamped], [@liskov2012viewstamped]

##### Practical Byzantine Fault Tolerance (PBFT)

[@castro1999practical]

<!-- 700BFT,  Query/Update (Q/U) and Hybrid Quorum (HQ) -->

##### Leaderless Byzantine State Machine Replication {#sec:leaderless}
SMR in general means that there is a single leader (as in Paxos and Raft...)...


Geo-replication... Due to their reliance on a leader replica, classical SMR protocols offer
limited scalability and availability under a geo-replicated setting. To solve this problem, recent protocols follow instead a leaderless approach, in which each replica is able
to make progress using a quorum of its peers. These new leaderless protocols
are complex and each one presents an ad-hoc approach to leaderlessness.

there are multi-leader approaches (TODO reference multi-leader raft) and most recent literature deals with leaderless SMR like Tempo [@enes2021tempo] or Wintermute [@rezende2021leaderless]...

Leaderless SMR shows its strengths especially when used in blockchains... high cost of replication... especially when strong consistent... see [Blockchain Consensus Protocols](#sec:blockchain-consensus)

##### Blockchain Consensus Protocols {#sec:blockchain-consensus}

In recent years, the problem of byzantine fault tolerant consensus has raised significantly more attention due to the widespread success of blockchains and blockchain-based applications, especially cryptocurrencies such as Bitcoin, which successfully solved the problem in a public setting without a central authority...

Blockchains are distributed systems par excellence... Using strong consistent consensus protocols to ensure every node reads the same... Handling not only fail-stop, but especially byzantine faults is crucial to those consensus protocols to secure a blockchain.

<!-- From https://tel.archives-ouvertes.fr/tel-03584254/document p24:

In more detail the Bitcoin protocol works as follows: Processes communicate
via reliable FIFO authenticated channels (implemented with TCP), thus modeling
a partially synchronous system [58]. When a transaction is sent by a client it
is placed in a shared pool, processes called miners collectively run a repeated
consensus protocol to select which transaction will be appended to the ledger.
In this consensus protocol, miners solve a cryptographic puzzle (proof-of-work)
to produce a valid block of transactions (i.e. miners are basically going through
a voting process, where they vote with their CPU power for valid blocks). This
proof-of-work procedure is computationally expensive (therefore, economically
expensive as well) to curb the effectiveness of Sybil attacks.
The advantage of using this approach is that it scales well with the numbers
of miners if one consider the system’s safety, since the more miners there are, the
more secure the service becomes (i.e. it gets increasingly more expensive to do
attacks). The downside is that Nakamoto’s consensus it is not strictly speaking
consensus. It can be the case that two miners concurrently solve the puzzle and
then append blocks to the blockchain in a way that neither block precedes the
other. This is called in the Bitcoin community as a fork. Typically, processes
continue to build on the block with the longest chain, that is, the one which has
more work done.
Dealing with forks highlight the disadvantages of the protocol, for instance: it
takes a waiting time of about 10 minutes to grow the chain by one block and from
an application point of view, waiting a few blocks is required (6 are recommended
in Bitcoin [26]) to guarantee that the transaction remains in the authoritative
chain. Therefore, the possibility of forks affects the consistency guarantees and
throughput of transactions (about 3 to 10 transactions per second in Bitcoin).
Many papers [11, 66, 110, 73] have been published with the intent of formalizing PoW protocols like Bitcoin and studying its consistency guarantees. In [66]
the authors provide one of the first formalizations of the Bitcoin protocol and
show that under synchronous environment assumptions the protocol guarantees
whp. an eventual consistent prefix. More broadly, in [11] the authors introduce the notion of Eventual Prefix and show that Bitcoin and other permissionless protocols abide to it. In [73, 72] the authors argue that Bitcoin implements
Monotonic Prefix Consistency whp.

-->

PoW (like the Nakamoto Consensus Algorithm that powers Bitcoin [@nakamoto2008bitcoin]), PoS, PoX, PBFT (see previous paragraph and [@li2020scalable])... all are BFT if conditions apply (especially, no 51% attack - remember, you need $2k + 1$ nodes to tolerate $k$ byzantine faulty nodes) (most important to prevent those failures!), BFT State Machine Replication protocols..

As blockchains typically have hundreds to hundred of thousands of replicas and need to provide high levels of consistency, classic single-leader state machine protocols are not applicable here, as they will dramatically slow down the whole chain due to their cost of replication.



TODO this is nice to have, remove if not enough time to mention this. If yes, reference (https://arxiv.org/pdf/1810.03357.pdf and https://arxiv.org/pdf/1904.04098.pdf)


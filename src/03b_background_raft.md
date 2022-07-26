## Raft, an Understandable Consensus Algorithm {#sec:raft}

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport}}

<!--  \cite{lamport1992distributed} -->

\todo{Fix citation reference in quotes/epigraphs}

This section first provides a general picture of the components and behavior of a Raft network as described by Ongaro
and Ousterhout in the release paper [@ongaro2013raft] and the dissertation [@ongaro2014consensus]...
In this work, we will focus on... we refer to the original paper, the dissertation and any related work on raft for...

The name: Raft because it is built on replicated logs, such as a raft. (TODO sketch it)

\todo{Use Raft outline from https://onlinelibrary.wiley.com/doi/pdf/10.1002/spe.3048}

- Based on distributed consensus, strong consistent
- "Raft provides linearizable consistency"
"Linearizability: a correctness condition for concurrent objects"
- TODO see and incorporate images of https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L13-strong-cap.pdf

- Raft is actually formally proven (TODO reference Proof GitHub Repo)

- TODO section on general raft properties
Suitable for ACID transactional systems...

TODO use cases and applications

TODO Raft is already used in event messaging and logging systems (Kafka, RabbitMQ), NoSQL object stores (MongoDB), distributed relational databases (CockroachDB), process orchestration systems (Camunda Zeebe), key-value stores (etcd), container orchestration (Kubernetes), smart grids [@sakic2017response]...

### Understandability

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable for all but a relatively small set of common tasks. Given how hard thinking is, if those intellectual tools are to succeed, they will have to substitute calculation for thought.}{--- \textup{Leslie Lamport}}

Why is there a need for another Consensus Protocol which offers equivalent fault-tolerance and performance characteristics as Paxos (does it? Cite proof here)? The main caveat of Paxos is the difficulty to understand the protocol, which makes it a challenge both for work in academia building on Paxos and for developers who want to implement the protocol and adapt it to their use case (see the [previous section on Paxos](#sec:paxos)). This inhibits progress in these areas and hampers discourse. In fact, Raft is a consensus protocol that was explicitly designed as an alternative to the Paxos family of algorithms to solve this understandability issues of how consensus can be achieved.

There are some other main differences in the protocol, which are handled in [Main Differences to Paxos](#sec:raft-vs-paxos).

TODO in essential the same class of protocol as Paxos (state machine consensus). It only adds understandability. In academia/literature, both are oftentimes referenced together or interchangeable when discussing consistency and consensus

... The author of raft even uses a visual and interactive game to teach the protocol (TODO link)
- Secret Lives of Data (show images and reference the page)

"Every system has a _complexity budget_: the system offers some benefits for its users, but if its complexity outweighs these benefits, then the system is no longer worthwhile."

"Raft appears almost uninteresting to academics. The academic community has not considered understandability per se to be an important contribution; they want novelty in some other dimension. Academia should be more open to work that bridges the gap between theory and practice. This
type of work may not bring any new functionality in theory, but it does give a larger number of students and practitioners a new capability, or at least substantially reduces their burden." [@ongaro2014consensus]

"Though my ideas and code solved the problems they were meant to address, they introduced an entirely new
set of problems: they would be difficult to explain, learn, maintain, and extend.
With Raft, we were intentionally intolerant of complexity and put that to good use. We set out
to address the inherently complex problem of distributed consensus with the most understandable
possible solution. Although this required managing a large amount of complexity, it worked towards
minimizing that complexity for others."

\todo{Refer and cite the raft user study}

### Main Differences to Paxos {#sec:raft-vs-paxos}

TODO

### Replicated State Machine

See [@garg2010implementing]

- Describe State Machine replication again very briefly and reference the full corresponding section from Background
    - State Machine replication is... For reference, see... in Raft, State Machine replication looks as follows...
- Describe State Machine in Raft

\todo{Show diagrams (redraw original paper diagrams of state machine)}

### Log Replication

\todo{Own subsection or part of "The Protocol in Detail"?}

- Describe Log replication in short, reference corresponding section from Background
    - Log replication is... For reference, see... in Raft, log replication looks as follows...

- Describe Raft Log
- show diagrams (redraw original paper diagrams of state machine)

- Raft therefore provides strong consistency (TODO reference to 03a, mention rule of the protocol that ensures this)

- TODO? write-ahead log https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html

\todo{Show diagrams (redraw original paper diagrams of state machine)}

### The Protocol in Detail

"Raft is linearizable (strong consistency) because it handles read/write all by the same leader" https://ieeexplore.ieee.org/document/9458806

TODO original raft is CP (in CAP) and PC/EC (in PACELC)

- Single leader: "Single master computing means somehow we order the changes." one factor to ensure strong consistency

- leader election, log replication, configuration changes, log compaction

- Messages
- Message clock synchonisation/enumeration with term:index (TODO reference to (#sec:consensus-protocols))
    - TODO: is it a logical clock? A hybrid logical clock (HLC)? https://medium.com/geekculture/all-things-clock-time-and-order-in-distributed-systems-hybrid-logical-clock-in-depth-7c645eb03682
- Random timeout 
\todo{Reference the random timeout thing from sec consensus-protocols which is one basic characteristic}
- Log compaction / snapshotting 
\todo{Reference the random timeout thing from 03a, which is one basic requirement for consensus protocols to fulfill the $\left \lceil (n + 1)/2 \right \rceil$ rule as shown}
- etc from paper

\todo{Show diagrams (redraw original paper diagrams of state machine)}

TODO ordering of events... [@lamport1978time]

TODO MTBF (TODO reference to 03a) - Leader election is the aspect of Raft where timing is most critical. Raft will be able to elect and maintain a steady leader as long as the system satisfies the following timing requirement: $\textrm{broadcastTime} \ll \textrm{electionTimeout} \ll \textrm{MTBF}$

> Should put this following sections here or in previous work?

TODO Network reconfiguration and fail-stop on faults: "Also in consensus protocols, shutting down a faulty node and initializing a fresh new one is effective as the data of the new node can be initialized using snapshotting in the background without impacting the whole cluster performance, as we show later in the corresponding section."

TODO network reconfig allows for rebalancing (cf. [@sec:partitioning]) without compromising availability

TODO partition-tolerance: Even while strong-consistent, raft also is ...

### Expected Dependability Properties

Lorem ipsum...

### Cost of Replication

In general, maintaining strong consistency is achievable but expensive (TODO reference 03a).

TODO why does raft still work?

The theoretical cost of replication (describes latency)... can be measured by the number of replicas... for a single state machine... network round trip... 

TODO rephrase

"Theoretically, the more the number of replicas, the higher the data availability; but the cost of replication (the detrimental in performance) increases at the same time. The challenge of the implementation is to achieve the optimal trade-off between the cost of replication and data access availability."

TODO some math here would be nice

The latency considerations...

low Latency defined as the latency less than the maximum wide-area
delay between replicas.

"Causal consistency is the strongest form of consistency that satisfies low Latency"

<!--
- Latency in decision making
- Low throughput
- System rigidity
- Consensus under load is tricky
-->

### Client Interaction

From Diss: "Unfortunately, when the consensus literature only addresses the communication between cluster servers,
it leaves these important issues out. We think this is a mistake. A complete system must interact
with clients correctly, or the level of consistency provided by the core consensus algorithm will go
to waste. As we’ve already seen in real Raft-based systems, client interaction can be a major source
of bugs, but we hope a better understanding of these issues can help prevent future problems."

TODO from diss: Session id to client requests, approaches to avoid possible duplicate requests (due tgo e.g. network problems causing no ack received by client), at-least-once semantics

### Possible Raft Extensions

This section discusses recently published Raft extensions, of which some are subject to academia, some others are implemented... that tackle different challenges and shortcomings of the original Raft proposal

#### Multi-Raft

- TODO descrive Weakness of Single-Raft

Raft does... but does not...

- TODO describe in short what Multi-Raft means
- Not covered in the originally proposed paper / dissertation
- Handled in multiple following papers and implementations
    - TODO reference/citation of multi-raft papers

- Allows partitioning (for load balancing and horizontal scalability), sharding

- TODO reference to partitioning section in background chapter

#### Byzantine Fault Tolerant Raft

TODO see [Types of Possible Faults](#sec:possible-faults) and [Consensus Protocols](#sec:consensus-protocols) for reference

There are certain approaches in research on byzantine fault tolerant versions and derivations of Raft... 
TODO reference/citation of [@clow2017byzantine] [@copeland2016tangaroa]...
Validation-Based Byzantine Fault Tolerant Raft [@tan2019vbbft]

#### Elasticity, Geo-Distribution and Auto-Scaling

The original Raft protocol does not provide lower latency to users by leveraging geo-replication due to its approach of a single leader at a time...

The leader can become a bottleneck that slows down the cluster and constitutes a single point of failure until the election of a new leader. 

See multi-raft and [@xu2019elastic]

#### Read Scalability

Having a single leader has also negative effects on read scalability in general... see [@arora2017leader]

TODO can we merge this and previous sub section?

#### Vertical Scalability

One single leader also does not scale with the number of cores and network cards on each machine [@deyerl2019search]

TODO if time: DepFast https://tianyin.github.io/pub/depfast-atc.pdf

#### Leaderless Raft

Lorem ipsum TODO find paper

#### Weakened Consistency Constraints

See https://ieeexplore.ieee.org/document/9458806


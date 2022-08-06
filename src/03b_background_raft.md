## Raft, an Understandable Consensus Algorithm {#sec:raft}

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport}}

<!--  \cite{lamport1992distributed} -->

\todo{Just reference the state machine section, and point out the raft specifics}

This section first provides a general picture of the components and behavior of a Raft network as described by Ongaro
and Ousterhout in the release paper [@ongaro2013raft] and the dissertation [@ongaro2014consensus]...
In this work, we will focus on... we refer to the original paper, the dissertation and any related work on raft for...

The name of this protocol is a metaphor and comes from the fact that it is built from replicated logs, much like a real raft is built from multiple wooden logs.

\todo{Use Raft outline from https://onlinelibrary.wiley.com/doi/pdf/10.1002/spe.3048}

- Based on distributed consensus, strong consistent
- "Raft provides linearizable consistency"
"Linearizability: a correctness condition for concurrent objects"
- TODO see and incorporate images of https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L13-strong-cap.pdf

- Raft is actually formally proven (TODO reference Proof GitHub Repo)

- TODO section on general raft properties
Suitable for ACID transactional systems...

TODO use cases and applications

TODO true for Kafka? Validate again!
TODO Raft is already used in event messaging and logging systems (Kafka, RabbitMQ), NoSQL object stores (MongoDB), distributed relational databases (CockroachDB), process orchestration systems (Camunda Zeebe), key-value stores (etcd), container orchestration (Kubernetes), smart grids [@sakic2017response]...

### Understandability

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable for all but a relatively small set of common tasks. Given how hard thinking is, if those intellectual tools are to succeed, they will have to substitute calculation for thought.}{--- \textup{Leslie Lamport}}

Why is there a need for another consensus protocol which offers equivalent fault-tolerance and performance characteristics as Paxos (does it? Cite proof here)? The main caveat of Paxos is the difficulty to understand the protocol, which makes it a challenge both for work in academia building on Paxos and for developers who want to implement the protocol and adapt it to their use case (see the [previous section on Paxos](#sec:paxos)). This inhibits progress in these areas and hampers discourse. In fact, Raft is a consensus protocol that was explicitly designed as an alternative to the Paxos family of algorithms to solve this understandability issues of how consensus can be achieved.

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

As another result to the user study, it is also easier to verify the correctness of the Raft algorithm than that of Multi-Paxos. Attention—This is the opinion of this works author: The Raft $\textrm{TLA}^{+}$ specification[^raft-tla] looks more understandable than the Paxos[^paxos-tla] and Multi-Paxos[^multi-paxos-tla] specification.

[^raft-tla]: https://github.com/ongardie/raft.tla/blob/master/raft.tla
[^paxos-tla]: https://github.com/tlaplus/tlapm/blob/main/examples/paxos/Paxos.tla
[^multi-paxos-tla]: https://github.com/DistAlgo/proofs/blob/master/multi-paxos/MultiPaxos.tla

### Main Differences to Paxos {#sec:raft-vs-paxos}

"Unlike Paxos, which is derived directly from the distributed consensus problem, Raft is proposed from the multi-replicated state machine. "

"https://www.alibabacloud.com/blog/paxos-raft-epaxos-how-has-distributed-consensus-technology-evolved_597127"

TODO

\paragraph{A Clear Definition of the Algorithm.}

While the multitude of different variants of the Paxos protocol family lack a single, clear definition of what to implement—especially for Multi-Paxos which is mostly derived from practical implementation in the software industry—Raft provides a complete and concise specification of every aspect of the protocol that can be easily followed to implement Raft in practice.

\paragraph{Consistent Logs.}

While logs in Paxos can have holes, Raft logs can't. Raft logs grow monotonically and append-only; their high-water mark shows to the latest committed log entry, and after that, they can differ in length, but always have a common prefix. In contrast, the Multi-Paxos log comes with nondeterminism; each log slot is decided independently which leads to holes in the logs in different places, which can block applications which require complete, consistent logs; the high-water mark in Paxos may move slower than the one in Raft.

\paragraph{No Learner Role.}

TODO

Having seperate nodes acting as learners only helps to distribute the replicated data, e.g. to increase data-locality for geo-replication, and to improve the read latency, without sacrificing write latency. 


<!--
\paragraph{Strong Single Leader}.

The concept of multiple proposers in Paxos introduces complexity and confusion, making the algorithm difficult to understand and implement. For many practical use cases, a single leader is often sufficient, so Raft is more suitable in these cases.

If two proposers keep preempting each other, no decision will be made

"Multi-Paxos uses only a very weak form of leadership as a performance optimization. "

There are still extensions to Multi-Paxos, describing a _distinguished leader_, which acts at the same time as a single distinguished proposer and learner, responsible for serving all client reads and writes, which is similar to Raft.

"Raft first elects a server as leader, then concentrates all decision-making onto the
leader. These two basic steps are relatively independent and form a better structure than Paxos,
whose components are hard to separate"
-->

\begin{table}[h!]
    \caption{Differences between Raft and Multi-Paxos}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X X} 
        \toprule
         & \thead{Raft} & \thead{Multi-Paxos} \\
        \midrule
        Leader & Strong & \makecell[l]{Weak; \\ many proposers allowed}  \\
        Log replication & Monotonicity guaranteed & Holes allowed \\
        Log submission & Linearized commits & Asynchronous commits \\
        Phases & \makecell[l]{Leader election \\ \& Log replication} & \makecell[l]{Prepare phase \\ \& Accept phase} \\
        \bottomrule
    \end{tabularx}
    \label{table:raft-vs-paxos}
\end{table}


### Replicated State Machine

See [@garg2010implementing]

- Describe State Machine replication again very briefly and reference the full corresponding section from Background
    - State Machine replication is... For reference, see... in Raft, State Machine replication looks as follows...
- Describe State Machine in Raft

\todo{Show diagrams (redraw original paper diagrams of state machine)}

Raft implements all important properties of state machine replication:

- Leader election,
- A generation clock (called a _term_),
- A log high-water mark,
- Log truncation,
- Snapshotting and state transfer,
- And network reconfigurations.

See subsection [@sec:state-machine-replication] for reference. The following subsections describe in detail what is specific to Raft only, and for everything else we refer to the descriptions in the [@sec:state-machine-replication] subsection.

### Log Replication

TODO remove it from here, as we already sketched this in state machine chapter

\todo{Own subsection or part of "The Protocol in Detail"?}

- Describe Log replication in short, reference corresponding section from Background
    - Log replication is... For reference, see... in Raft, log replication looks as follows...

- Describe Raft Log
- show diagrams (redraw original paper diagrams of state machine)

- Raft therefore provides strong consistency (TODO reference to 03a, mention rule of the protocol that ensures this)

- TODO? write-ahead log https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html

\todo{Show diagrams (redraw original paper diagrams of state machine)}

"It is important to ensure that all the cluster nodes receive all the log entries from the leader, even when they are disconnected or they crash and come back up. Raft has a mechanism to make sure all the cluster nodes receive all the log entries from the leader."

https://martinfowler.com/articles/patterns-of-distributed-systems/replicated-log.html

### The Protocol in Detail

"Raft is linearizable (strong consistency) because it handles read/write all by the same leader" https://ieeexplore.ieee.org/document/9458806

TODO original raft is CP (in CAP) and PC/EC (in PACELC)

- Single leader: "Single master computing means somehow we order the changes." one factor to ensure strong consistency

- leader election, log replication, configuration changes, log compaction

- Messages
- Message clock synchonisation/enumeration with term:index (TODO reference to (#sec:state-machine-replication))
    - A Generation Clock "Raft uses the concept of a Term for marking the leader generation."
    - TODO: is it a logical clock? A hybrid logical clock (HLC)? https://medium.com/geekculture/all-things-clock-time-and-order-in-distributed-systems-hybrid-logical-clock-in-depth-7c645eb03682
    - Random timeout 

    
In Raft, the FLP impossibility problem is circumvented by adding pseudo-synchronous behavior: all the messaging goes in rounds, where each rounds time is bounded by using timeouts. The rounds are enumerated using terms and index. If during the timeout period no entries are appended to a node (an empty AppendEntries command acts as a plain heartbeat), this node assumes there is a problem with the leader and starts a new voting round (denoted by a new term).


\todo{Reference the random timeout thing from sec consensus-protocols which is one basic characteristic}
- High-water mark ( In the Raft consensus algorithm, high-water mark is called 'CommitIndex'.)
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

#### Node Roles and The Voting Phase

\paragraph{Follower.}

\paragraph{Candidate.}

\paragraph{Leader.}

\paragraph{The Voting Phase.}

When a node joins the cluster, it joins as a follower. After a random timeout, it starts an election by voting for itself, turning into a candidate. To add pseudo-synchronous behavior (to mitigate the FLP impossibility problem), it will restart the election again if no leader was decided after another timeout. Every new election introduces a new term, closing the previous one. If a candidate node receives votes from a quorum (an absolute majority of the nodes), it immediately turns into a leader, sending out heartbeats to notify all other nodes of the successful vote. If due to network latency, partitioning or other problems a new leader node discovers any node with a higher term then the own one, it turns back into a follower to potentially start a new vote for a new term, to restore consistency. The same applies to candidates. If any candidate receives a heartbeat from a newly assigned leader, it turns back into a follower. This ensures that there is only one leader for any given term—at least if there is no network partition. The whole voting phase and the transitions are illustrated in a state diagram in figure \ref{fig:raft-transitions}.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-transitions.pdf}
  \caption[Transitions between roles of a cluster node in Raft]{Transitions between roles of a cluster node in Raft. When a node joins the cluster, it starts as a follower. After a random timeout, it starts an election by voting for itself, turning into a candidate. To add pseudo-synchronous behavior to mitigate the FLP impossibility problem, it starts a new election after the timeout. Every new election introduces a new term, closing the previous one.}
  \label{fig:raft-transitions}
\end{figure}

#### Raft Log

\begin{figure}[h]
  \centering
  \includegraphics[width=0.35\textwidth]{images/raft-log-entry-anatomy.pdf}
  \caption[Anatomy of a Raft log entry]{Anatomy of a Raft log entry. A Raft log entry contains the term and the index of its creation, as well as the command payload.}
  \label{fig:raft-log-entry-anatomy}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-log-committed-entries.pdf}
  \caption[Anatomy of a Raft log entry]{Raft logs of different nodes of a cluster. Every log entry contains the current term and index as well as its command. Entries are committed if they have been acknowledged by a quorum, thus safe to be applied to the state.}
  \label{fig:raft-log-committed-entries}
\end{figure}

"An entry is considered _committed_ if it is safe for that entry to be applied to state machines"

\paragraph{Snapshotting.}

"In the snapshot system, if the state Sn in the state machine at a certain time is safely applied to most of the nodes,
then Sn is considered safe, and all the states previous to Sn can be discarded, therefore
the initial operating state S0 is steadily changed to Sn, and other nodes only need to
obtain the log sequence starting from Sn when obtaining logs."

Also see figure X of subsection Y


### Formal Verification

<!--
Paxos has been formally verified by Lamport using the $\textrm{TLA}^{+}$ formal specification language (see subsection [@sec:cost-of-replication] for reference)[@lamport2006fast]; Multi-Paxos has also been formally verified by Chand et al. [@chand2016formal].
-->

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

From file:///Users/christian.konrad/Documents/Paper/style%20inspiration/OngaroPhD.pdf:

"Raft’s performance is similar to other consensus algorithms such as Multi-Paxos. The most important case for performance is when an established leader is replicating new log entries. Raft achieves
this using the minimal number of messages (a single round-trip from the leader to half the cluster).
It is also possible to further improve Raft’s performance. For example, Raft easily supports batching and pipelining requests for higher throughput and lower latency, as described below. Chapter 11
discusses various other optimizations that have been proposed in the literature for other algorithms;
many of these could be applied to Raft, but we leave this to future work.
Figure 10.2(a) shows the steps Raft must take to process a client’s request. Typically, the most
time-consuming steps are writing the new log entry to disk and replicating it across the network.
Writing to disk can take anywhere from 100 µs for a fast solid state disk to 10 ms for a slow magnetic
disk, while the latencies of today’s networks can vary from 5 µs round trip times in highly optimized
datacenter networks to 400 ms round trip times for networks that span the globe. In our experiments
on a local area network, either the disk or the network dominated, depending on which model of
solid state disk we used."

https://static-content.springer.com/pdf/chp%3A10.1007%2F978-981-19-2456-9_44.pdf?token=1659198387340--f9759c58b2485e870e0abb1d9b3fff01f85132273c5c3141e42a4e5c13d2d05ae6a668cf3b88663be5152ca5aafb20c00cc9fded348adac788a70c5a721ca775 [@li2022improved]

"Raft’s linear semantics causes client requests to eventually turn into an execution
sequence that is received, executed, and submitted sequentially, regardless of the concurrency levels of requests. Under a large number of concurrent requests, two problems will
arise. 1. The Leader must process the proposal under the Raft mechanism, so the Leader
is a performance bottleneck. 2. The processing rate is much slower than the request rate.
A large number of requests will cause a large number of logs to accumulate and occupy
bandwidth for a long time and memory."

"Problem 1 can be solved with the Multi-Raft-Group [4]. Mutil-Raft regards a Raft
cluster as a consensus group. Each consensus group will generate a leader. Different
leaders manage different log shards. In this way, the Leader’s load pressure will be
evenly divided among all consensus groups, thus preventing the Raft cluster’s single
Leader from becoming an obstacle. "

TODO solved by sharding. Reference this in system design: That's what we did

#### Deciding on the Number of Replica Nodes

https://martinfowler.com/articles/patterns-of-distributed-systems/quorum.html

"The cluster can function only if majority of servers are up and running. In systems doing data replication, there are two things to consider:

The throughput of write operations.
Every time data is written to the cluster, it needs to be copied to multiple servers. Every additional server adds some overhead to complete this write. The latency of data write is directly proportional to the number of servers forming the quorum. As we will see below, doubling the number of servers in a cluster will reduce throughput to half of the value for the original cluster.

The number of failures which need to be tolerated.
The number of server failures tolerated is dependent on the size of the cluster. But just adding one more server to an existing cluster doesn't always give more fault tolerance: adding one server to a three server cluster doesn't increase failure tolerance."

TODO table from https://martinfowler.com/articles/patterns-of-distributed-systems/quorum.html

TODO throughput graph from raft diss p 145 (163 on pdf)
file:///Users/christian.konrad/Documents/Paper/style%20inspiration/OngaroPhD.pdf

### Client Interaction

From Diss: "Unfortunately, when the consensus literature only addresses the communication between cluster servers,
it leaves these important issues out. We think this is a mistake. A complete system must interact
with clients correctly, or the level of consistency provided by the core consensus algorithm will go
to waste. As we’ve already seen in real Raft-based systems, client interaction can be a major source
of bugs, but we hope a better understanding of these issues can help prevent future problems."

TODO from diss: Session id to client requests, approaches to avoid possible duplicate requests (due tgo e.g. network problems causing no ack received by client), at-least-once semantics

### Possible Raft Extensions {#sec:raft-extensions}

This section discusses recently published Raft extensions, of which some are subject to academia, some others are implemented... that tackle different challenges and shortcomings of the original Raft proposal

#### Asynchronous Batch Processing

https://static-content.springer.com/pdf/chp%3A10.1007%2F978-981-19-2456-9_44.pdf?token=1659198387340--f9759c58b2485e870e0abb1d9b3fff01f85132273c5c3141e42a4e5c13d2d05ae6a668cf3b88663be5152ca5aafb20c00cc9fded348adac788a70c5a721ca775

TODO reference original raft premises

The original Raft protocol has been designed pessimisticly, but formally correct, in the face of unreliable network behavior where the network communication between clusters is not reliable and are susceptible to packet loss, delay, network jitter, etc. "But in practice, the communication between computers tends to be stable most of the time (that is, the delay between nodes is much less than the time
of a Heartbeat). In addition, general reliable communication protocols such as TCP have
a retransmission mechanism, with which lost packets will be retransmitted immediately,
so it is possible to recover in a short time even if there is a failure. Therefore, we can
change the second premise to: _the computer network is not always in a dangerous state._" [@li2022improved]

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

#### Learner Roles 

One of the most popular services that implements Raft, etcd (which backs Kubernetes), added the concept of learners, as of Paxos, to Raft: https://etcd.io/docs/v3.3/learning/learner/

#### Elasticity, Geo-Distribution and Auto-Scaling

The original Raft protocol does not provide lower latency to users by leveraging geo-replication due to its approach of a single leader at a time...

The leader can become a bottleneck that slows down the cluster and constitutes a single point of failure until the election of a new leader. 

See multi-raft and [@xu2019elastic]

Also learner roles can help

#### Read Scalability

Having a single leader has also negative effects on read scalability in general... see [@arora2017leader]

TODO can we merge this and previous sub section?

#### Vertical Scalability

One single leader also does not scale with the number of cores and network cards on each machine [@deyerl2019search]

TODO if time: DepFast https://tianyin.github.io/pub/depfast-atc.pdf

#### Leaderless Raft

"Raft leverages its strong leader for understandability and reducing mechanism, and this key design
choice is at odds with reducing the leader’s involvement in normal operations. Thus, if Raft were
modified to support these optimizations, the end result would differ considerably from the Raft
algorithm, and it would probably be significantly harder to understand."

https://arxiv.org/abs/2008.02512
https://tel.archives-ouvertes.fr/tel-03584254/document
https://www.sciencedirect.com/science/article/pii/S1383762122000844

#### Weakened Consistency Constraints

See https://ieeexplore.ieee.org/document/9458806


## Raft, an Understandable Consensus Algorithm {#sec:raft}

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport}}

In this subchapter, we present the Raft consensus algorithm[^raft-name]. First, we explain the motivation behind Raft. Second, we provide a general overview of the components and behavior of a Raft network as described by Ongaro and Ousterhout in the paper [@ongaro2014raft] and the dissertation [@ongaro2014consensus]. The protocol is then explained in detail. Since Raft belongs to the family of state machine replication, we refer to section [@sec:state-machine-replication] for the basics of the protocol. Finally, we discuss some recent extensions of the Raft protocol.

[^raft-name]: The name of this protocol is a metaphor and is derived from the fact that it is built out of replicated logs, much like a real raft is built from multiple wooden logs.

<!--
- Based on distributed consensus, strong consistent (linearizability)
- TODO see and incorporate images of https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L13-strong-cap.pdf
- Raft is actually formally proven (TODO reference Proof GitHub Repo) TODO TLA+
-->

Nowadays, Raft is used in many production systems, such as 

- Permissioned blockchains (e.g., Hyperledger Fabric, see subsection [@sec:blockchain-consensus]), 
- Event and message brokers (e.g., Kafka and RabbitMQ, see subsection [@sec:kafka]), 
- NoSQL object stores (e.g., MongoDB [@zhou2021fault]),
- Distributed relational databases (e.g., CockroachDB [@cockroach2022]),
- Process orchestration systems (e.g., Camunda Zeebe [@camunda2022zeebe]),
- Key-value stores (e.g., etcd [@etcd2022]), 
- Container orchestration (e.g., Kubernetes [@kubernetes2022github]), 
- smart grids [@sakic2017response] and many more.

### Understandability

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable for all but a relatively small set of common tasks. Given how hard thinking is, if those intellectual tools are to succeed, they will have to substitute calculation for thought.}{--- \textup{Leslie Lamport}}

Why is there a need for another consensus protocol which offers similar fault-tolerance and performance characteristics as Paxos (see subsection [@sec:paxos])? The Raft consensus protocol was presented in 2013 by Diego Ongaro in reaction to the complexity of Paxos. Paxos was (and still often is) the default protocol taught in lectures at universities when it comes to consensus protocols. The authors of Raft observed that despite its popularity, Paxos is an algorithm that is difficult to understand and implement, especially since all official publications about Paxos lack a complete description of the protocol. We also discovered and described the latter in subsection [@sec:paxos]. This inhibits progress in these areas and hampers discourse. Raft is a consensus protocol that was explicitly designed as an alternative to the Paxos family of algorithms to solve this understandability issues of how consensus can be achieved.

There are some other main differences to Paxos and Viewstamped Replication in the protocol, which are handled in sections [@sec:raft-vs-paxos] and [@sec:raft-vs-viewstamped].

In essential, Raft belongs to the same class of protocols as Paxos, as well as Viewstamped Replication (see subsection [@sec:viewstamped]): state machine replication. But Raft adds understandability to make consensus available to a broader audience. Since complexity is often a reason why research or development projects are slowed down or even canceled, Ongora picks up the term _complexity budget_: "Every system has a _complexity budget_: the system offers some benefits for its users, but if its complexity outweighs these benefits, then the system is no longer worthwhile" [@ongaro2014consensus]. This saying, the complexity of an algorithm or protocol limits how much of it can fit into the heads of academic researchers, distributed systems designers, and engineers. In his dissertation, Ongora also expresses criticism of the academic community to "not consider[$\dots$] understandability per se to be an important contribution". To support understandability and to teach the protocol, the Raft authors even implemented an interactive visualization of the Raft consensus protocol, called _The Secret Lives of Data_ [@raft2013viz].

The understandability was evaluated in a user study and the results of this study confirmed that Raft is more understandable as Paxos. The study compared the answers of students to quiz questions about Raft and Paxos after they learned each algorithm. We refer to the dissertation for the results [@ongaro2014consensus]. As another result to the user study, it is also easier to verify the correctness of the Raft algorithm than that of Multi-Paxos. Attention—This is the opinion of the author of this work: The Raft $\textrm{TLA}^{+}$ specification[^raft-tla] looks more understandable than the Paxos[^paxos-tla] and Multi-Paxos[^multi-paxos-tla] specifications.

[^raft-tla]: https://github.com/ongardie/raft.tla/blob/master/raft.tla
[^paxos-tla]: https://github.com/tlaplus/tlapm/blob/main/examples/paxos/Paxos.tla
[^multi-paxos-tla]: https://github.com/DistAlgo/proofs/blob/master/multi-paxos/MultiPaxos.tla


<!--
"With Raft, we were intentionally intolerant of complexity and put that to good use. We set out
to address the inherently complex problem of distributed consensus with the most understandable
possible solution. Although this required managing a large amount of complexity, it worked towards
minimizing that complexity for others."
-->

### Main Differences to Paxos {#sec:raft-vs-paxos}

The main differences between Raft and Paxos are listed in this section.

<!-- "https://www.alibabacloud.com/blog/paxos-raft-epaxos-how-has-distributed-consensus-technology-evolved_597127" -->

##### Complexity.

We have already laid out the differences in complexity between the two protocols.

##### The Problem Perspective.

While Paxos originated from the distributed consensus problem, Raft originates directly from the perspective of a replicated state machine. The distributed consensus problem focuses on agreeing to a single value. Multi-Paxos is still derived from this perspective, making it difficult to understand and argue about the ordering guarantees of Paxos for multiple operations. In contrast, strongly consistent ordering sits in the core of the Raft consensus protocol, making it easy to understand and reason about.

##### A Clear Definition of the Algorithm.

While the multitude of different variants of the Paxos protocol family lack a single, clear definition of what to implement—especially for Multi-Paxos which is mostly derived from practical implementation in the software industry—Raft provides a complete and concise specification of every aspect of the protocol that can be easily followed to implement Raft in practice.

##### Consistent Logs.

While logs in Paxos can have holes, Raft logs can't. Raft logs grow monotonically and append-only; their high-water mark shows to the latest committed log entry, and after that, they can differ in length, but always have a common prefix. In contrast, the Multi-Paxos log comes with nondeterminism; each log slot is decided independently which leads to holes in the logs in different places, which can block applications which require complete, consistent logs; the high-water mark in Paxos may move slower than the one in Raft.

##### Less Complex Roles.

In Raft, there is no seperate learner role. Each node takes only one role at a time, while in Paxos, a node can have all possible combination of the three roles. This reduces complexity of every system running Raft, since there are no varying network topologies.

This comes with the sacrifice of having seperate nodes acting solely as learners to help distributing the replicated data, e.g. to increase data-locality for geo-replication, and to improve the read latency, without sacrificing write latency. 

##### Strong Single Leader.

The concept of multiple proposers in Paxos introduces complexity and confusion, making the algorithm difficult to understand and implement. For many practical use cases, a single leader is often sufficient, so Raft is more suitable in these cases. Raft elects one node to be the leader. Subsequently, all requests go through the leader. This simplifies the management of the replicated log. These two basic protocol steps are separate, whereas in Paxos these complementary protocol steps are intertwined.

But, there are also extensions to Multi-Paxos, describing a _distinguished leader_, which acts at the same time as a single distinguished proposer and learner, responsible for serving all client reads and writes, which is similar to Raft.

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

### Main Differences to Viewstamped Replication {#sec:raft-vs-viewstamped}

Similar to the previous section, this section shows the main differences between Raft and Viewstamped Replication (VR, see subsection [@sec:viewstamped]).

##### Complexity.

Viewstamped Replication is probably the protocol with a specification closest to that of Raft. But still, it is more complex than Raft. Especially since Raft provides strong leader properties and minimizes the functionality in non-leaders, its protocol is more compact and less complex than that of VR, and also offers a more complete specification: the Raft protocol only uses 4 different message types, whereas VR needs 10 to perform the same tasks.

##### Leader Election.

Raft guarantees that if a leader is elected, its log contains all of the committed entries, making it easier to reason about the Raft logs in the cluster. In Raft, log entries only flow in one direction: from leaders to followers. Additionally, leaders will never overwrite existing entries in their own logs. Compared to that, VR maintains additional protocol steps to identify missing log entries in primary nodes and to transmit them to the new primary, which adds complexity both in the protocol and in intermediate cluster states. There is also a difference in the way a leader is voted for: in VR, a node accepts a voting request using a deterministic function of the term number, while in Raft, a node accepts a voting request in a non-deterministic way by accepting the vote of the first node who asks.

##### Transferring Logs.

Raft uses term-index pairs to denote the current state of the replicated logs and to test for consistency, whereas VR transfers entire logs, such as in leader election. 

##### Primary-Copy.

Viewstamped Replication originates from the Primary-Copy approach, while Raft puts the log first. This means that in Viewstamped Replication, state changes are replicated rather than the original commands.

For a more detailed explanation of the differences, we refer to the dissertation [@ongaro2014consensus] and this discussion with Ongaro in the Raft-Dev mailing group [@raft2015mailinggroup].

### Protocol Overview

The Raft protocol implements a replicated state machine by managing a replicated log. As a result, Raft provides strong consistency. Raft is a protocol that is as concise as possible without lacking correctness and including all properties that are important in state machine replication. It divides the problem of multi-value consensus into three relatively distinct subproblems:

- Leader election,
- Log replication,
- And safety (cf. section [@sec:safety-reliability] for a definition of safety).

We will outline these three subproblems throughout the next sections. For a full overview of the protocol, we refer to the the original paper [@ongaro2014raft] and the dissertation [@ongaro2014consensus]. We also extracted the summary page of the protocol from the original paper into the appendix of this work.

##### Consistency.

Raft is linearizable, thus providing strong consistency, since it handles all reads and writes by a single, strong leader. From a trade-off perspective, Raft is a CP class (in CAP) respectively PC/EC class protocol (in PACELC).

##### Replicated State Machine.

We presented the state machine replication approach in subsection [@sec:state-machine-replication]. Raft implements all important properties of state machine replication:

- Leader election,
- A generation clock (called a _term_),
- A high-water mark for the replicated log (called the `lastApplied` term-index),
- Log truncation,
- Snapshotting and state transfer,
- And network reconfigurations (cluster membership changes).

The following subsections describe what is specific to Raft only, and for everything else we refer to the descriptions in subsection [@sec:state-machine-replication].

##### Messaging.

The basic Raft protocol gets by with just two messages: `appendEntries` and `requestVote`. Raft also supports log compaction and cluster membership changes with two additional messages: `installSnapshot` and `add/removeServer`.

### Node Roles {#sec:node-roles}

In Raft, a node has one of three roles, and only exactly one role at the same time. Figure \ref{fig:raft-transitions} shows these roles and how a node transitions between them.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-transitions.pdf}
  \caption[Transitions between roles of a cluster node in Raft]{Transitions between roles of a cluster node in Raft. When a node joins the cluster, it starts as a follower. After a random timeout, it starts an election by voting for itself, turning into a candidate. To add pseudo-synchronous behavior to mitigate the FLP impossibility problem, it starts a new election after the timeout. Every new election introduces a new term, closing the previous one.}
  \label{fig:raft-transitions}
\end{figure}

##### Follower.

All nodes begin in the follower role on Raft cluster start up or if a new node joins the cluster. A follower listens to `appendEntries` requests from the leader. If such a request contains log entries, it applies them to its log. It also commits all entries in its log that are older then the `lastApplied` term-index pair in this message. It also interprets `appendEntries` requests as heartbeats from the leader. Therefore, it can also receive `appendEntries` requests without log entries in the payload. Once it does not receive a heartbeat from the leader after a certain _election timeout_ period, it assumes the leader node crashed (failure detection), transitions into a candidate, and starts a leader election.

##### Candidate.

As a candidate, a node votes for itself and sends `requestVote` messages to all other nodes in the cluster. If it receives granted votes from a majority of votes of the total cluster (including itself), it transitions into the next leader.

##### Leader.

For a Raft cluster, only one single leader is allowed at a time. The leader is the only node that is allowed to respond to client requests, whether they are read or write requests. The leader sends `appendEntries` requests to the followers to replicate the write requests they receive from clients. When on idle, the leader continues to send `appendEntries` with an empty payload which acts as a heartbeat.

When the network is partitioned, a split-brain situation can emerge. Only in this situation, there can be more than one leader, since both partitions won't know if the other is still alive and assumes the nodes of it do be crashed. The Raft protocol ensures that after a cluster recovers from partitioning, a single leader is elected and strong consistency is restored again across all nodes, which may result in loss of data at one of the partitions.

### Guarantees

Raft comes with several guarantees, that in turn guarantee liveness, safety and strong consistency of the protocol. These guarantees are given by the definition of the log replication and leader election steps, which we will present in the subsequent sections. The guarantees have been proven in the Raft dissertation [@ongaro2014consensus], and can be verified using the Raft $\textrm{TLA}^{+}$ specification. We reproduce these guarantees in their original form here.

\begin{table}[H]
    \caption{Raft guarantees, as described by Ongaro and Ousterhout}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X} 
        \toprule
        Election Safety & At most one leader can be elected in a given term. \\
        Leader Append-Only & A leader never overwrites or deletes entries in its log; it only appends new entries. \\
        Log Matching & If two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index. \\
        Leader Completeness & If a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms. \\
        State Machine Safety & If a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index. \\
        \bottomrule
    \end{tabularx}
    \label{table:raft-guarantees}
\end{table}

### Log Replication

In Raft, a log entry contains the write command that is to be replicated and applied to the state machine, the current term and the index of this log entry. This is illustrated in figure \ref{fig:raft-log-entry-anatomy}. The term is a monotonically growing generation clock (cf. subsection [@sec:state-machine-replication]), representing the current phase of an incumbent leader. With every leader election, a new term is started, increasing the term number. The term number is crucial in providing safety during log replication and leader election in the face of failures. If a follower node that is in a higher term receives messages from a lower term, it rejects these messages and requests a new voting.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.3\textwidth]{images/raft-log-entry-anatomy.pdf}
  \caption[Anatomy of a Raft log entry]{Anatomy of a Raft log entry. A Raft log entry contains the term and the index of its creation, as well as the command payload.}
  \label{fig:raft-log-entry-anatomy}
\end{figure}

Every node maintains a `lastApplied` and `commitIndex` term-index pair. The `lastApplied` term-index pair denotes the last log entry that was applied to the state machine, while the `commitIndex` denotes the last log entry that is known to be _committed_. An entry is considered committed once it can be safely applied to state machines (i.e., it is known to a node quorum). Hence, the `commitIndex` implements the high-water mark of state machine replication.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/raft-log-committed-entries.pdf}
  \caption[Anatomy of a Raft log entry]{Raft logs of different nodes of a cluster. Every log entry contains the current term and index as well as its command. Entries are committed if they have been acknowledged by a quorum, thus safe to be applied to the state.}
  \label{fig:raft-log-committed-entries}
\end{figure}

In figure \ref{fig:raft-replication}, the replication of a log entry is illustrated[^fowler-illustration] as it happens in the case of normal operation.

[^fowler-illustration]: The illustration is based on a blog post by Martin Fowler: https://martinfowler.com/articles/patterns-of-distributed-systems/replicated-log.html

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-replication.pdf}
  \caption[Illustration of the log replication in Raft]{Simplified illustration of the log replication in Raft, based on Martin Fowler}
  \label{fig:raft-replication}
\end{figure}

A client sends a write request to a Raft cluster of three nodes. All nodes are already in a consistent state, containing the same entries in their logs. Once the write request arrives at the cluster leader (Node A), it appends it to its own log. Subsequently, it requests the other nodes to append this entry by sending a `appendEntries` message. This message contains the entries to append (here, a single entry), as well as the current term of the leader and the last `commitIndex`.

The followers receiving this message compares the term and `commitIndex` with their own logs. In case one follower contains more log entries from the same term, it rejects the request and starts a new vote. If it contains uncommitted entries from an older term—which could have happened due to network partitioning—it removes them from its log and continues accepting the append request. If the term in the message is lower than the one of the follower, the follower rejects and notifies the leader, which steps back into the follower role. In the case of the illustration, everything is ok, so the followers accept the request and append the entry to their logs. 

The followers acknowledge the append. Once the leader receives acknowledgements from a quorum (including itself), it increases its `commitIndex` to the highest term-index pair of the acknowledged entries. It is now safe to append the entries to the state machine. In the next `appendEntries` request—in this case a heartbeat—it sends the updated `commitIndex`. The followers then update their own `commitIndex`, which also allows them to append the previously replicated log entries.

This process ensures a global execution ordering of commands to the replicated state machines.

### Leader Election

We described the node roles in section [@sec:node-roles]. When a node joins the cluster, it joins as a follower. After a random election timeout, it starts an election by voting for itself, turning into a candidate. To add pseudo-synchronous behavior (to mitigate the FLP impossibility problem), it will restart the election again if no leader was decided after another random timeout. The randomness is important here to avoid deadlocks and to introduce nondeterminism, which is also required due to the FLP problem (cf. paragraph [@sec:flp-impossibility]). Leader election is the part of the Raft protocol where timing is most critical. The leader election timeout is to be chosen in a way that in general, after a successful vote, heartbeats are sent before other followers time out and start a vote. This is reflect in the following timing requirement:

$$\mathtt{broadcastTime} \ll \mathtt{electionTimeout} \ll \textrm{MTBF}$$

Here, the `broadcastTime` is the average time it takes to send messages to all nodes in the cluster and receive their responses, and MTBF is the Mean Time Between Failures (cf. section [@sec:scalability]). 

Every new election introduces a new term, closing the previous one. This increases the term number. If a candidate node receives votes from a quorum (an absolute majority of the nodes), including its own vote, it immediately turns into a leader, sending out heartbeats to notify all other nodes of the successful vote. If due to network latency, partitioning or other problems a new leader node discovers any node with a higher term then the own one, it turns back into a follower to potentially start a new vote for a new term, to restore consistency. The same applies to candidates. If any candidate receives a heartbeat from a newly assigned leader, it turns back into a follower. This ensures that there is only one leader for any given term—at least if there is no network partition[^leader-election-viewstamped]. The way Raft is designed ensures that a quorum always knows the current committed log entries. This results in a guarantee that during voting, a candidate will not turn into a leader unless its log contains all committed entries. We illustrate the voting phase by an example in figures \ref{fig:raft-leader-election}—\ref{fig:raft-leader-election-3}. 

[^leader-election-viewstamped]: This leader election protocol is similar to the view change protocol of Viewstamped Peplication, where a term is called a view.

In figure \ref{fig:raft-leader-election}, nodes D and E crashed. Node E was the leader in term 1. Since there is no leader, nodes A—C do not receive a `appendEntries` message anymore. Node B times out, turns into a candidate and starts a leader election for a new term, voting for itself and sending `requestVote` messages to the other nodes. The `requestVote` message contains the current term and the last known term-index of node B. Since it contains less entries than Node A, its vote is rejected by A. Node B does not receive a grant from a quorum (at least 3 nodes, since the original cluster contained 5 nodes) and hence loses this vote, transitioning back into a follower.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-leader-election.pdf}
  \caption[Failed leader election in Raft]{Illustration of the leader election in Raft after a node failure. A node requests a vote, but fails the vote since it does not receive a grant from a majority of cluster nodes (including itself).}
  \label{fig:raft-leader-election}
\end{figure}

After the failed election, node A times out and starts a new one. It increases the term again, now to 3, votes for itself and requests votes. Since it contains all known log entries and there are no other issues, it wins the election, turning into the new leader for term 3. It then announces its new leadership by sending `requestVote` messages to the other followers, preventing them from starting a new vote, and also sending the remaining log entry in the payload of the message. This entry is then appended to the logs of the followers.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-leader-election-2.pdf}
  \caption[Successful leader election in Raft]{A successful leader election in Raft. A node requests a vote and receives a grant from a majority of cluster nodes.}
  \label{fig:raft-leader-election-2}
\end{figure}

Node A replicated the second entry from term 1 to nodes B and C. Since this entry is from a older term, the entry is not committed after this replication. To commit this entry and apply it to the state machine, a no-operation is appended, replicated and committed in the next heartbeat to update all Raft logs to the newly started term.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/raft-leader-election-3.pdf}
  \caption[Committing the remainder of the log after leader election]{After a successful leader election, the remaining uncommitted log entries are committed. Since the Raft protocol forbids committing entries of an old term, a no-operation is appended and replicated to update all logs to the newest term.}
  \label{fig:raft-leader-election-3}
\end{figure}

### Log Compaction

Raft allows for log compaction by creating snapshots of resulting states from committed prefixes of the log, following what we outlined in subsection [@sec:state-machine-replication]. For this, Raft introduces another message, called `installSnapshot`. We refer to the extended version of the original paper for details on this message and the compaction protocol [@ongaro2014raft].

### Network Reconfiguration / Cluster Membership Changes

We discussed Raft from the perspective of a fixed cluster configuration. In practice, clusters are often reconfigured during their lifetime. Events that result in a change in cluster membership include upscaling outdated machines, replacing crashed servers, outscaling when adding new partitions, rebalancing when a node has proportionally too much load, or increasing or decreasing the replication factor. In some cases, such as in containerized environments, it may be cheaper and more convenient to discard a crashed node and immediately boot a new node with a state snapshot, rather than rebooting a failed node. Raft supports this conveniently and natively, compared to other replication protocols. We won't explain this in detail here and refer to the extended version of the original paper [@ongaro2014raft].

### Cost of Replication

In theory, maintaining strong consistency is achievable but expensive. Raft is used extensibly in production systems, including those that manage a lot of data with very high throughput. We will show some of those throughout later chapters of this work.

In general, the performance of Raft is similar to other consensus algorithms, such as Multi-Paxos. The theoretical cost of replication of Raft can be measured by the number of replicas. It is based on the network roundtrip and the I/O of the cluster nodes to write the log entries to disk. When we consider the best case as illustrated in figure \ref{fig:raft-replication}—ignoring the case of failures and leader election—we can estimate the additional latency per write by a single additional round-trip from a leader to the half of the nodes with the lowest inter-cluster latency. A very slow node won't affect the overall latency since only a majority of nodes is needed to confirm a write. But, we also need to incorporate additional I/O overhead of the Raft log. In the Raft dissertation, a FIFO buffer is suggested to put in place between the raft log and the replication engine, so multiple threads at the leader can take care of writing to the log and sending `appendEntries` requests in parallel.

There are some extensions to Raft that aim to improve latency for reads, writes or both. They can only reduce latency to a certain degree, since there is a physical limit of disk speed and the speed of light. We will outline them in section [@sec:raft-extensions].

The challenge of every Raft implementation is to achieve the optimal trade-off between the cost of replication and data access availability. For even more effective latency improvements, the basic idea is to reduce the number of Raft protocol instances, if possible, meaning that raft log entries are sent in batches, reducing the overall number of network roundtrips dramatically. This is especially useful in wide-area networks, while it poses the risk of data loss once the leader fails. We use a comparable approach in our implementation in section [@sec:buffered-inserts].

<!--
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
-->

### Client Interaction {#sec:raft-client-interaction}

Other than most consensus algorithms, the Raft dissertation also covers how clients should interact with the cluster for lowest possible latency. It explains how clients can discover a cluster and its current leader, how to avoid possible duplicate requests du to lost intermediate acknowledgements, and also how to linearize writes in case one drops the leader property for read requests. We want carry this topic further here, but refer to corresponding chapter 6 in the dissertation [@ongaro2014consensus].

### Possible Raft Extensions {#sec:raft-extensions}

To conclude this subchapter, this section outlines recently published Raft extensions that tackle different challenges and shortcomings of the original Raft proposal. It should be noted that many of these extensions add further complexity to the protocol, which is contradictory to its initial purpose of providing understandability.

#### Multi-Raft

Multi-Raft is the most important extension of Raft and has been implemented in most production systems. With Multi-Raft, the concept of _Raft groups_ is introduced to allow to scale with the cluster load. This comes in particularly handy when partitioning, where each Raft group manages a single partition. Each Raft group contains their own leader and manages their own state machine instance.  This helps preventing that a single leader for the whole cluster would become a bottleneck by evenly distributing the write load among nodes. Multi-Raft additionally optimizes heartbeat behavior by allowing a node that is the leader in many Raft groups to send a single `appendEntries` message for all groups, instead of one message per group. There is no dedicated literature that describes Multi-Raft, though. Most of its descriptions come from actual implementations.

#### Asynchronous Batch Processing

Li et al. recently suggested to reduce the latency of Raft with the introduction of asynchronous batch processing in Raft [@li2022improved]. They introduce a new _proposal_ mechanism, allowing clients to asynchronously propose writes, which are then processed in batches in the consensus and replication engine. This works under the assumption that network failures are rare, relying on retransmission mechanisms of the underlying network transport protocol (for instance, TCP).

#### Byzantine Fault Tolerant Raft

TODO see [Types of Possible Faults](#sec:possible-faults) and [Consensus Protocols](#sec:consensus-protocols) for reference

There are certain approaches in research on byzantine fault tolerant versions and derivations of Raft. Since we do not focus on byzantine fault-tolerance, we only list them here for reference. Notable ones include Tangaroa [@copeland2016tangaroa], Validation-Based Byzantine Fault Tolerant Raft [@tan2019vbbft] and this nameless one [@clow2017byzantine]. Some of them are used in blockchains.

#### Learner Roles 

One of the most popular services that implements Raft, etcd (which backs Kubernetes), added the concept of learners, similar to that of Paxos, to Raft [@etcd2021learners]. They introduced learners to decrease read latency and to put load of from the leader. With increasing load, the leader is more likely to delay the delivery of heartbeats, and therefore followers are more likely to trigger leader elections when they shouldn't, which can lead to serious problems. 

#### Read Scalability

Similar to the concept of learner roles, Arora et al. propose to allo for _quorum reads_, which means allowing followers that succesfully applied the latest committed entries of the leader to answer read requests as well [@arora2017leader].

#### Elasticity, Geo-Distribution and Auto-Scaling

The original Raft protocol does not provide lower latency to users by leveraging geo-replication due to its property of having a single leader at a time. With multi-Raft and partitioning, this can be mitigated to some level. But there is also literature on how to make Raft truly elastic by leveraging its cluster reconfiguration capabilities, supporting auto-scaling and geo-distribution. Xu et al. introduces _Geo-Raft_ to extend Multi-Raft with two additional node roles: _secretaries_ which takes log processing for the leader and _observers_ which process read requests for followers [@xu2019elastic].

#### Vertical Scalability

Deyerl et al. propose a Raft-based replication protocol called _Niagara_ in which the process of appending new log entries is parallelized across multiple Raft instances [@deyerl2019search]. It allows a Raft node to scale vertically with the number of cores and network cards on the machine.

#### Weakened Consistency Constraints

Wang et al. propose to rethink the consistency requirements when using Raft for key-value stores [@wang2021rethink]. The authors claim that some of the constraints are not necessary to have consistency for distributed key-value stores. This is related to other approaches we have discussed for key-value stores, such as chain replication or optimistic replication.

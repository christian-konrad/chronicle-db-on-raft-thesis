## Raft, an Understandable Consensus Algorithm {#sec:raft}

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport} \cite{lamport1992distributed}}

\todo{Fix citation reference in quotes/epigraphs}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. [@ongaro2013raft] [@ongaro2014consensus]

- TODO not only reference paper, but also diss

- Based on distributed consensus, strong consistent

- Raft is actually formally proven (TODO reference Proof GitHub Repo)


- TODO section on general raft properties
Suitable for ACID transactional systems...

### Understandability

Why is there a need for another Consensus Protocol which offers equivalent fault-tolerance and performance characteristics as Paxos (does it? Cite proof here)? The main caveat of Paxos is the difficulty to understand the protocol, which makes it a challenge both for work in academia building on Paxos and for developers who want to implement the protocol and adapt it to their use case (see the [previous section on Paxos](#sec:paxos)). This inhibits progress in these areas and hampers discourse. In fact, Raft is a consensus protocol that was explicitly designed as an alternative to the Paxos family of algorithms to solve this understandability issues of how consensus can be achieved.

There are some other main differences in the protocol, which are handled in [Main Differences to Paxos](#sec:raft-vs-paxos).

... The author of raft even uses a visual and interactive game to teach the protocol (TODO link)

### Main Differences to Paxos {#sec:raft-vs-paxos}

TODO

### Replicated State Machine

See [@schneider1990implementing] [@garg2010implementing]

- Describe State Machine replication in short, reference corresponding section from Background
    - State Machine replication is... For reference, see... in Raft, State Machine replication looks as follows...
- Describe State Machine in Raft
- show diagrams (redraw original paper diagrams of state machine)

### Log Replication

- Describe Log replication in short, reference corresponding section from Background
    - Log replication is... For reference, see... in Raft, log replication looks as follows...

- Describe Raft Log
- show diagrams (redraw original paper diagrams of state machine)

- Raft therefore provides strong consistency (TODO reference to 03a, mention rule of the protocol that ensures this)

- TODO? write-ahead log https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html

### The Protocol in Detail

- Messages
- etc from paper

TODO ordering of events... [@lamport1978time]

> Should put this following sections here or in previous work?

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

### Multi-Raft

- TODO descrive Weakness of Single-Raft

Raft does... but does not...

- TODO describe in short what Multi-Raft means
- Not covered in the originally proposed paper / dissertation
- Handled in multiple following papers and implementations
    - TODO reference/citation of multi-raft papers

- Allows partitioning (for load balancing and horizontal scalability), sharding

- TODO reference to partitioning section in background chapter

### Byzantine Fault Tolerant Raft

TODO rephrase
The Byzantine Generals Problem is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]

There are certain approaches in research on byzantine fault tolerant versions and derivations of Raft... 
TODO reference/citation of [@clow2017byzantine] [@copeland2016tangaroa]...
Validation-Based Byzantine Failure Tolerant Raft [@tan2019vbbft]

### Elasticity, Geo-Distribution and Auto-Scaling

The original Raft protocol does not provide lower latency to users by leveraging geo-replication due to its approach of a single leader at a time...

See multi-raft and [@xu2019elastic]

### Read Scalability

Having a single leader has also negative effects on read scalability in general... see [@arora2017leader]

TODO can we merge this and previous sub section?

### Vertical Scalability

One single leader also does not scale with the number of cores and network cards on each machine [@deyerl2019search]

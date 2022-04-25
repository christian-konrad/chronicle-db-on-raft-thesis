## Raft, an Understandable Consensus Algorithm

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport} \cite{lamport1992distributed}}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. [@ongaro2013raft] [@ongaro2014consensus]

- TODO not only reference paper, but also diss

- Raft is actually formally proven (TODO reference Proof GitHub Repo)


### Understandability

- Why another Consensus Protocol and not Paxos?

... The author of raft even uses a visual and interactive game to teach the protocol (TODO link)

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

- TODO? write-ahead log https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html

### The Protocol in Detail

- Messages
- etc from paper

> Should put this following sections here or in previous work?

### Multi-Raft

- TODO descrive Weakness of Single-Raft

Raft does... but does not...

- TODO describe in short what Multi-Raft means
- Not covered in the originally proposed paper / dissertation
- Handled in multiple following papers and implementations
    - TODO reference/citation of multi-raft papers

- Allows partitioning (for load balancing and horizontal scalability), sharding

### Byzantine Fault Tolerant Raft

TODO rephrase
The Byzantine Generals Problem is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]

There are certain approaches in research on byzantine fault tolerant versions and derivations of Raft... 
TODO reference/citation of [@clow2017byzantine] [@copeland2016tangaroa]...
Validation-Based Byzantine Failure Tolerant Raft [@tan2019vbbft]

### Elasticity, Geo-Distribution and Auto-Scaling

See [@xu2019elastic]

### Read Scalability

The original Raft protocol knows only one leader at a time (ref here)... for both reads and writes...
This has negative effects on read scalability... see [@arora2017leader]

### Vertical Scalability

One single leader also does not scale with the number of cores and network cards on each machine [@deyerl2019search]

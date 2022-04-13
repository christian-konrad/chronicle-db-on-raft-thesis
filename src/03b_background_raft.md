## Raft, an Understandable Consensus Algorithm

\epigraph{There is a race between the increasing complexity of the systems we build and our ability to develop intellectual tools for understanding their complexity. 
If the race is won by our tools, then systems will eventually become easier to use and more reliable. If not, they will continue to become harder to use and less reliable [...].}{--- \textup{Leslie Lamport} \cite{lamport1992distributed}}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. [@ongaro2013raft]

- TODO not only reference paper, but also diss

- Raft is actually formally proven (TODO reference Proof GitHub Repo)


### Understandability

- Why another Consensus Protocol and not Paxos?

### Replicated State Machine

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

### Multi-Raft

- TODO descrive Weakness of Single-Raft

Raft does... but does not...

- TODO describe in short what Multi-Raft means
- Not covered in the originally proposed paper / dissertation
- Handled in multiple following papers and implementations
    - TODO reference/citation of multi-raft papers

- Allows partitioning (for load balancing and horizontal scalability), sharding

### Byzantine Fault Tolerant Raft

- TODO reference/citation of [@clow2017byzantine, @copeland2016tangaroa]

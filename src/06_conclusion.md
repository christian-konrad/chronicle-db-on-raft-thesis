# Conclusion {#sec:conclusion}

- TODO conclude and start a discussion

<!-- Style in this way:

This paper presented UNISTORE, the first fault-tolerant and
scalable data store that combines causal and strong consistency. UNISTORE carefully integrates state-of-the-art scalable
protocols and extends them in nontrivial ways. To maintain
liveness despite data center failures, unlike previous work,
UNISTORE commits a strong transaction only when all its
causal dependencies are uniform. Our results show that UNISTORE combines causal and strong consistency effectively:
3.7× lower latency on average than a strongly consistent system with 1.2ms latency on average for causal transactions. We
expect that the key ideas in UNISTORE will pave the way for
practical systems that combine causal and strong consistency
 -->

- Has negative impact on performance (within the expectations of strong consistency) when scaling vertically
- Can have a positive impact when scaling horizontally, when partitioning is leveraged in a smart way
- Is fault-tolerant as Raft...
- Raft formally proven
    - If ratis is 100% following raft protocol and therefore formally correct still to be confirmed
- Possible to build a strong-consistent event store with great availability
- TODO



The given implementation of the Raft replication protocol has a high cost of replication. With a growing number of replica nodes, the throughput decreases. There are various strategies to address this issue: partitioning strategies (TODO ref to section),... to mention a few. They all come with some level of tradeoffs.

## Recommendations

> TODO depends on outcome. If throughput decreases to much, will recommend to weigh between throughput and availability

## Future Work

\epigraph{Strive for progress, not perfection.}

### Horizontal Scalability

#### Sharding/Partitioning

- Per time split; Log merge...
- Allows to increase number of nodes without decreasing throughput
- Show diagram of further partitioning/sharding techniques (i.e. by timesplit..., reference methods from background chapter here)

#### Elasticity: Auto Scaling and Recovery

- Using Kubernetes
- Bring it together with Kubernetes Replicas

#### Geo-Replication

- In it's given implementation not suitable for geo-replication (TODO reference to 03b) due to it's single-leader characteristic
- There are different solution proposals for that like multi-raft, multi-leader raft... (TODO reference those from 03b)
- Or one could learn from strategies of protocols with weaker consistency [@hsu2021cost] 

#### More Raft Optimizations
- A paper describes a formal mapping of Paxos optimization to Raft with guaranteed correctness... [@wang2019parallels]

### Distributed Queries

- Reference to popular databases
- JOINs between streams
- Queries over sharded time splits

### Distributed Storages

- Hadoop / HDFS
- S3

### gRPC and Protocol Buffers API

- Performance Boost
- Protobuf must be married with ChronicleDB schemas
- May infer the one from the other
- Or may consider Avro as the source of truth for both?

### User Friendliness

- One single RaftServer per Node
    - Get rid of addtional port for management quorum
- Code generator for messages
    - Creates operationMessages, clients and executor bodies from protos
    - To reduce time to value and potential implementation errors
- Provide as library
    - Developers can include it to their spring boot apps to quickly develop and provide replicated data services

### Mising Implementations

- Exception Handling, Graceful shutdown

\pagebreak

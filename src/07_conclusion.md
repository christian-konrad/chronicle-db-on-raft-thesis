# Conclusion {#sec:conclusion}

- Has negative impact on performance when scaling vertically
- Positive impact when scaling horizontally
- Raft formally proven
    - If ratis is 100% following raft protocol and therefore formally correct still to be confirmed
- Possible to build a strong-consistent event store with great availability
- TODO

## Recommendations

> TODO depends on outcome. If throughput decreases to much, will recommend to weigh between throughput and availability

## Future Work

\epigraph{Strive for progress, not perfection.}

### Horizontal Scalability

#### Sharding/Partitioning

- Per time split; Log merge...
- Allows to increase number of nodes without decreasing throughput

#### Elasticity: Auto Scaling and Recovery

- Using Kubernetes
- Bring it together with Kubernetes Replicas

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

# Conclusion {#sec:conclusion}

\epigraph{All software architectures capture tradeoffs, and deciding between alternatives is about choosing which tradeoffs you can live with in a particular context.}{--- \textup{Jimmy Lin}}

\todo{Conclude and start a discussion}

This work discussed... and presented ChronicleDB on a Raft...

With strong consistency, we arrive somehow at a one-size-fits-all solution. It works in every case, but not perfectly. The trade-offs in latency and availability may make this approach not suitable enough for extremely high-throughput cases, but there are multiple other strategies to mitigate this, such as partitioning, which we will introduce in our implementation. It may sound pessimisticly, but you can't have it all: the CAP and PACELC theorem have proven that in theory, and many implementations have shown that in practice. But actually, it is not that bad at all: even if there will always be a decision to be made between generalized vs. specialized implementations, both offer better dependability and scalability properties than a single-node deployment may ever be possible to serve.

TODO if we allow for OOO, it is still not suitable for real-time applications with high business criticality, but this is already the case for standalone ChronicleDB (but chance for OOO is higher in distributed Chronicle)

We have shown this in the evaluation: ...

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

 
<!--
Also write a discussion:
Interpretations: what do the results mean?
Implications: why do the results matter?
Limitations: what can’t the results tell us?
Recommendations: what practical actions or scientific studies should follow?


Summarise your key findings
Start this chapter by reiterating your problem statement and research questions and concisely summarising your major findings. Don’t just repeat all the data you have already reported – aim for a clear statement of the overall result that directly answers your main research question. This should be no more than one paragraph.

Examples
The results indicate that…
The study demonstrates a correlation between…
The analysis confirms…
The data suggests that…

https://www.scribbr.co.uk/thesis-dissertation/discussion/
https://www.scribbr.co.uk/thesis-dissertation/conclusion/
-->

- Has negative impact on performance (within the expectations of strong consistency) when scaling vertically
- Can have a positive impact when scaling horizontally, when partitioning is leveraged in a smart way
- Is fault-tolerant as Raft...
- Raft formally proven
    - If ratis is 100 % following raft protocol and therefore formally correct still to be confirmed
- Possible to build a strong-consistent event store with great availability
- TODO serialization expensive (and what else?)

The given implementation of the Raft replication protocol has a high cost of replication. With a growing number of replica nodes, the throughput decreases. There are various strategies to address this issue: partitioning strategies (TODO ref to section),... to mention a few. They all come with some level of trade-offs.

## Recommendations and Future Work

\epigraph{\hfill Strive for progress, not perfection.}{}

<!--
We refer to Gotsman et al. [@gotsman2016cause] to help deciding for a consistency model. If low latency is important, we strongly advice system architects to think about their event design and apply practices of _domain-driven-design_ (DDD), such as breaking down their application to trivial facts (domain events) and derived aggregates and categorizing them as monotonic or non-monotonic, as described in subsection [@sec:consistency-decisions]. Afterwards, the appropriate consistency model can be decided upon, which further guarantees safety and liveness for the adapted system design. If latency is not that important or can be mitigated in other ways through sharding and other practices, we recommend opting for strong consistency, as this ensures safety in any case and allows starting with a simple system design that can be better thought out afterwards. Regarding out-of-order events, they must be either disallowed to provide strong consistency, or the consistency level must be explicitly flagged as eventual consistency, even if a strongly consistent replication protocol is used. We also recommend to look at time-bound partial consistency, as it can help with out-of-order events. We hope that this work will serve as a guidance here.
-->


### Consistency Considerations

- As other related data stores such as InfluxDB show, having all data replicated strong consistently with a consensus protocol may be inpractical and slows down the system/currently this is challenging to provide both strong consistency and support high throughputs
- As others are doing (Apache IoTDB), one could use different protocols for different parts of the implementation, as such for meta data/cluster management, region management, schemas, data itself... Maybe just use raft for meta data and have primary-copy or another approach for the data itself? Perhaps the user should decide this for themselves based on their consistency requirements?
- But also others like EventStoreDB do provide strong consistency on all layers, to the cost of... 

\todo{Evaluate performance of EventStoreDB!}

TODO the final consistency model should be decided on
- use case
- requirements of the programmer: Complexity of the API (dist sys renders itself like a single node app or do they need to work around stale data?)
- write vs read heavyness
- infrastructure capabilities and mitigation possibilities (sharding possible, network partitioning resilience, latency improvements...)

We recommend that you should try to go for strong consistency, apply different discussed techniques (sharding/partitioning, partial replication... etc) and then work your way down the consistency levels until your system satisfies the practical (!) requirements, as theoretical considerations often do not manifest in real world applications. TODO cite again many of the strong consistent but HA and low latency systems we have discussed throughout the work

TODO out-of-order: Violates strong consistency (not from the point of view of the event store, but from the point of view of client applications expecting a reliable stream of events); maybe allow to choose two consistency levels: strong consistency with exceptional out-of-order and "actual strong consistency"?

#### Multiple Consistency Levels

"A compromise is to allow multiple consistency levels to coexist in the data store." [@bravo2021unistore]

- reference CAP and PACELC and describe our trade-offs, and if we could allow users to configure the trade-offs themselvesor 
- We could limit consensus to critical parts, like creating and evolving an event schema. (All clients agree to simultaneously make the change.), updating meta data and the quorum itself
- We could also just let database user (= developer) decide
- We have seen both approaches in other db vendors, as well as fully strong consistently replicated dbs
- Partial replication and geo-replication
- If serving data to only a portion of users, not all data need to be replicated into all locations
- Metadata (book keeper) always strong consistent
- But let user decide on consistency level for data
- Similar to Kafka
- So it is suitable to architect an edge-cloud system design with multiple different consistency levels to fit the edge levels best 

- Recommendation: Allow for different consistency levels on a per-stream basis.


<!-- TODO CRDTs do not seem to be suitable. Redis say they can be used for Multi-region IoT data ingest, but I think this will slow the event store down too much by idempotency and so on. But what about SECROs?
And: Can't we transform the whole ChronicleDB TAB+ tree into a CRDT like with causal trees? http://archagon.net/blog/2018/03/24/data-laced-with-history/#causal-trees-in-depth Can we make all operations idempotent and commutative? 
---
It could also be considered to implement the event store as an CRDT instead of applying consensus, limiting its consistency to strong eventual consistency. This implementation would be feasible to be used on the edge in edge-cloud networks, beeing able to ingest high volumes of events while providing high availability, while feeding them into a central cluster and merging them. But in this case, the overall performance of the event store will be compromised, as this requires out-of-order merging. Efficient merging strategies need to be discovered to allow for such a CRDT implementation.
-->

### Horizontal Scalability

There are also other strategies to improve the throughput that do not violate correctness and consistency guarantees...

#### Sharding/Partitioning

- Per time split; Log merge... 
    - Like in IoTDB: "data partitions can be defined according to both time slice and time-series ID" [@wang2020iotdb]
- Allows to increase number of nodes without decreasing throughput
- Show diagram of further partitioning/sharding techniques (i.e. by timesplit..., reference methods from background chapter here)

TODO refer to system design; sharding of a single stream

- TODO if there will ever be a concept of transactions here, adjusting the protocol the be coordination-free may be suitable, as with Eris, to avoid high latency by transaction and replication roundtrips, but it requires control of the infrastructure (i.e., the network layer)

#### Elasticity: Auto Scaling and Recovery

- Multi-leader etc., TODO reference to the raft extensions section
- Using Kubernetes
- Bring it together with Kubernetes Replicas
- What about failure detection in general?
- Does it make sense to move failure detection to K18n? Would this just add a new layer of dependency on a management process, that we initially wanted to avoid?

#### Geo-Replication

Providing strong consistency with geo-replication is expected to be too expensive for high-throughput event stores, as shown in subsection [sec:geo-replication]. Depending on the locations of the geo-replicas, it will limit the maximum throughput dramatically. Batching writes, i.e. by using a buffer, can help, but only to a certain extend. 
We recommend to invest time into investigating modern geo-replication approaches, e.g. such with weaker consistency models and coordination-free variants, justified to the cost of consistency. Note that not all use cases allow for this. In that cases, we suggest to look into improved sharding techniques or multi-leader or leaderless Raft, or just mirroring the data into read-only replicas (which maintain the correct event order).

[@hsu2021cost] 

### Raft Optimizations

- A paper describes a formal mapping of Paxos optimization to Raft with guaranteed correctness... [@wang2019parallels]

For Raft improvements and extensions in general, see subsection [@sec:raft-extensions].

#### Raft Log Implementation

\todo{mention here AND in system design}
TODO the log is a very naive implementation. Popular applications even use efficient embedded (in-memory?) storage engines such as RocksDB (like CockroachDB https://github.com/cockroachdb/cockroach/issues/38322) - and they also come with WAL logs, so we have a multi-layer architecture of the raft log. Even if this comes with some issues, we can learn from it and use a better log approach. Our naive approach (the Ratis default one) can slow down the system. Makes sense to have the raft log running on a different thread to not block other operations and improve I/O

TODO even consider log-less replication, to satisfy the "The log is the database" thing again, as described in [@skrzypzcak2020towards]
 In contrast to the log-based architecture, replicas agree on a sequence of replica states instead of a sequence of commands

 "the leader writes the log entry to its disk in parallel
with replicating the entry to the followers, which can reduce latency significantly"

"Transient raft log; after succesful commit, entries in log are gone; index is always latest snapshot. Also marry raft log and OOO write-ahead log. If we need to derive the log again: with OOO, not possible"

#### Buffer Improvements

The traditional Raft algorithm executes a client’s request to meet linear consistency with sequential execution and sequential submission, which badly impacts performance.
As a possible extension, Li et al. propopsed an additional asynchronous batch processing approach [@li2022improved] that increases the performance of raft in the face of concurrent requests, which in their experimental setup improved the raft performance by 2 to 3.6 times.

#### Introduce Learner Roles

A weakness of Raft compared to Paxos is the absence of the learner role. By introducing dedicated learner nodes, read latency can be improved by providing data-locality for certain shards. This comes especially handy for operating on the edge: It allows to provide read replicas on the edge, close to the clients, without compromising write latency.

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

### Framework Choices

At the time of this writing, the Hashicorp Raft implementation in Go [https://github.com/hashicorp/raft] is the most popular, reliable and supported one... due to the characteristics of the Go language and ecosystem which make Go a perfect fit for challenges in distributed systems, there is a strong advice to use this implementation if you want to start with a proven solution... plug'n'play (is it?)... extensible (is it?)

Apache Ratis has support problems... Java makes it somehow inefficient (does it?), but it is very extensible and customizable (big plus) and fits 1:1 in your Java environment (as for ChronicleDB and Spring Boot)...

### User Friendliness

- One single RaftServer per Node
    - Get rid of addtional port for management quorum
- Code generator for messages
    - Creates operationMessages, clients and executor bodies from protos
    - To reduce time to value and potential implementation errors
- Provide as library
    - Developers can include it to their spring boot apps to quickly develop and provide replicated data services

### Further Missing Implementations

- Exception Handling, Graceful shutdown...

\pagebreak

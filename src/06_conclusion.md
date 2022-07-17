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
- TODO



The given implementation of the Raft replication protocol has a high cost of replication. With a growing number of replica nodes, the throughput decreases. There are various strategies to address this issue: partitioning strategies (TODO ref to section),... to mention a few. They all come with some level of trade-offs.

## Recommendations

> TODO depends on outcome. If throughput decreases to much, will recommend to weigh between throughput and availability

## Future Work

\epigraph{Strive for progress, not perfection.}


### Consistency Considerations

- As in InfluxDB, having all data replicated with a consensus protocol may be inpractical and slows down the system
- As others are doing (Apache IoTDB), one could use different protocols for different parts of the implementation, as such for meta data/cluster management, region management, schemas, data itself... Maybe just use raft for meta data and have primary-copy or another approach for the data itself? Perhaps the user should decide this for themselves based on their consistency requirements?

TODO the final consistency model should be decided on
- use case
- requirements of the programmer: Complexity of the API (dist sys renders itself like a single node app or do they need to work around stale data?)
- write vs read heavyness
- infrastructure capabilities and mitigation possibilities (sharding possible, network partitioning resilience, latency improvements...)

We recommend that you should try to go for strong consistency, apply different discussed techniques (sharding/partitioning, partial replication... etc) and then work your way down the consistency levels until your system satisfies the practical (!) requirements, as theoretical considerations often do not manifest in real world applications. TODO cite again many of the strong consistent but HA and low latency systems we have discussed throughout the work

#### Multiple Consistency Levels

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

#### Elasticity: Auto Scaling and Recovery

- Multi-leader etc., TODO reference to the raft extensions section
- Using Kubernetes
- Bring it together with Kubernetes Replicas
- What about failure detection in general?
- Does it make sense to move failure detection to K18n? Would this just add a new layer of dependency on a management process, that we initially wanted to avoid?

#### Geo-Replication

- In it's given implementation not suitable for geo-replication (TODO reference to 03b) due to it's single-leader characteristic
- But can be simply adressed using data mirroring 
- ? There are different solution proposals for that like multi-raft, multi-leader raft... (TODO reference those from 03b)
- ? Or one could learn from strategies of protocols with weaker consistency [@hsu2021cost] 

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

### Mising Implementations

- Exception Handling, Graceful shutdown

\pagebreak

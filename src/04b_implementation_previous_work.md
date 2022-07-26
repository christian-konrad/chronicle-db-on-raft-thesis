## Previous Implementations {#sec:previous-work}

\epigraph{If I have seen further it is by standing on the shoulders of Giants.}{--- \textup{Isaac Newton}}

- ... as far as current research goes into theoretical details of what is possible with Raft, seeing it in action gives insights into practical usage and therefore current limits and possibilities...

### Criteria for Review and Comparison

TODO everything that narrows the scope here

#### Why not a Leaderless Protocol?

- TODO reference to (#sec:leaderless)
- Not crucial for replication for fault-tolerance
- ChronicleDB is distributed, but not decentralized
- Only a small replication factor needed compared to massive georeplicated systems (blockchains etc.)
- Understandability favorable over extreme replica node scalability

### Popular Applications of Replication Protocols

TODO from [@gotsman2016cause]: Tools that allow the user to choose the consistency level
Large-scale distributed systems often rely on replicated databases
that allow a programmer to request different data consistency guarantees for different operations, and thereby control their performance. Using such databases is far from trivial: requesting stronger
consistency in too many places may hurt performance, and requesting it in too few places may violate correctness.

in general, NoSQL is mostly eventual (but see mongodb as a counterexample)

\todo{Add all the others here, but prepare them in excel for convenience}
\todo{May split in metadata and payload consistencvy}
\todo{Also consider this categorizing https://www.researchgate.net/profile/Michael-Lang-28/publication/282761553/figure/tbl1/AS:668236437790738@1536331390709/Representative-System-Services-Categorized-Through-the-Taxonomy.png}

Also look at https://www.researchgate.net/figure/Consistency-models-comparison_tbl1_331104869

\begin{table}[h!]
    \caption{The list of systems studied in this work, and their primary dependability and consistency properties}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}l X X X} 
        \toprule
        \thead{System} & \thead{Consistency Model} & \thead{Data Level} & \thead{Replication Protocol} \\
        \midrule
        LogCabin & Strong & All (but small amounts) & Raft \\
        Apache Kafka & User Choice & All (? Meta Strong + Data Eventual?) & Raft-like \\
        Zookeeper & Strong & All (It keeps metadata only) & Primary-Copy \\
        MongoDB & Strong & TODO & Pull-Based Consensus (Raft-like) \\
        Redis & Eventual, Strong is WIP & All & CRDTs, Master-Replica, Raft is WIP \\
        AAA & - & - \\
        \bottomrule
    \end{tabularx}    
    \label{table:systems-studied}
\end{table}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/CAP-dbs.pdf}
  \caption{Illustration of the CAP theorem}
  \label{fig:cap-dbs}
\end{figure}

TODO many of these systems allow their users to decide for the consistency model and therefore for the availability-consistency (and the latency-consistency, considering the PACELC theorem) trade-off.

#### LogCabin

Written by the author of Raft... [@ongaro2015logcabin]
... this thesis' implementation for metadata and cluster management follows similar principle... see Implementation Section 

#### Apache Kafka

##### Zookeeper

- Is Primary-Copy Replication (TODO refer to 03a)
- External agent
- Provides a risk as a single point of failure
- Ends up in Primary-Secondary Replication
- Discontinued in KIP-500: Self-Managed Meta Quorum with Raft, see next section
- TODO provide the reasoning from Apache here why Zookeeper is considered bad vs. raft/self-organized quorum
- Zookeeper is used in some/many (?) other Apache Projects:
- https://mesos.apache.org/ 

##### KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum (Raft)

- TODO summarize [@kafka2022kip500]

- Kind-of Raft

- Consistency level can be chosen by user

<!--

Unclean leader election: What if they all die?
Note that Kafka's guarantee with respect to data loss is predicated on at least one replica remaining in sync. If all the nodes replicating a partition die, this guarantee no longer holds.
However a practical system needs to do something reasonable when all the replicas die. If you are unlucky enough to have this occur, it is important to consider what will happen. There are two behaviors that could be implemented:

Wait for a replica in the ISR to come back to life and choose this replica as the leader (hopefully it still has all its data).
Choose the first replica (not necessarily in the ISR) that comes back to life as the leader.
This is a simple trade-off between availability and consistency. If we wait for replicas in the ISR, then we will remain unavailable as long as those replicas are down. If such replicas were destroyed or their data was lost, then we are permanently down. If, on the other hand, a non-in-sync replica comes back to life and we allow it to become leader, then its log becomes the source of truth even though it is not guaranteed to have every committed message. By default from version 0.11.0.0, Kafka chooses the first strategy and favor waiting for a consistent replica. This behavior can be changed using configuration property unclean.leader.election.enable, to support use cases where uptime is preferable to consistency.

This dilemma is not specific to Kafka. It exists in any quorum-based scheme. For example in a majority voting scheme, if a majority of servers suffer a permanent failure, then you must either choose to lose 100 % of your data or violate consistency by taking what remains on an existing server as your new source of truth.

Availability and Durability Guarantees
When writing to Kafka, producers can choose whether they wait for the message to be acknowledged by 0,1 or all (-1) replicas. Note that "acknowledgement by all replicas" does not guarantee that the full set of assigned replicas have received the message. By default, when acks=all, acknowledgement happens as soon as all the current in-sync replicas have received the message. For example, if a topic is configured with only two replicas and one fails (i.e., only one in sync replica remains), then writes that specify acks=all will succeed. However, these writes could be lost if the remaining replica also fails. Although this ensures maximum availability of the partition, this behavior may be undesirable to some users who prefer durability over availability. Therefore, we provide two topic-level configurations that can be used to prefer message durability over availability:
Disable unclean leader election - if all replicas become unavailable, then the partition will remain unavailable until the most recent leader becomes available again. This effectively prefers unavailability over the risk of message loss. See the previous section on Unclean Leader Election for clarification.
Specify a minimum ISR size - the partition will only accept writes if the size of the ISR is above a certain minimum, in order to prevent the loss of messages that were written to just a single replica, which subsequently becomes unavailable. This setting only takes effect if the producer uses acks=all and guarantees that the message will be acknowledged by at least this many in-sync replicas. This setting offers a trade-off between consistency and availability. A higher setting for minimum ISR size guarantees better consistency since the message is guaranteed to be written to more replicas which reduces the probability that it will be lost. However, it reduces availability since the partition will be unavailable for writes if the number of in-sync replicas drops below the minimum threshold.

-->

#### CockroachDB

- Raft
- NewSQL: Offering benefits of distributed systems with scalable OLTP while still providing ACID capability for relational databases [@pavlo2016s]

- Problem with WAL on raft log https://github.com/cockroachdb/cockroach/issues/38322

#### Camunda Zeebe

- TODO add stuff from our pages
- https://github.com/camunda/zeebe/tree/main/raft/src/main/java/io/zeebe/raft/state
- docs... etc

#### RabbitMQ

- Raft in Quorum Queues [@rabbitmq2021quorum]
- https://www.rabbitmq.com/quorum-queues.html
- modern queue type for RabbitMQ implementing a durable, replicated FIFO queue
- available as of RabbitMQ 3.8.0
- built for a set of use cases where data safety is a top priority
- They are not designed to be used for every problem. Their intended use is for topologies where queues exist for a long time and are critical to certain aspects of system operation, therefore fault tolerance and data safety is more important than, say, lowest possible latency and advanced queue features.

#### ElasticSearch

- Some-kind-of Raft

#### Google Spanner

https://static.googleusercontent.com/media/research.google.com/de//pubs/archive/45855.pdf

Paxos
External consistency: 
- similar to linearizability
- "Spanner’s external consistency invariant is that for any two
transactions, T1 and T2 (even if on opposite sides of the globe):
if T2 starts to commit after T1 finishes committing, then the timestamp for T2 is greater than the timestamp for T1."  


Cloud Spanner is semantically indistinguishable from a single-machine database. 
Uses TrueTime
built on Googles infrastructure, therefore even with strong consistency, it is highly available and can even use realtime timestamps. This is only possible due to the fact that Google is in full control of the infrastructure. "Even though it provides such strong guarantees, Cloud Spanner enables applications to achieve performance comparable to databases that provide weaker guarantees (in return for higher performance)."

"Spanner uses the Paxos algorithm as part of its operation to shard (partition) data across up to hundreds of servers.[1] It makes heavy use of hardware-assisted clock synchronization using GPS clocks and atomic clocks to ensure global consistency.[1] TrueTime is the brand name for Google's distributed cloud infrastructure, which provides Spanner with the ability to generate monotonically increasing timestamps in datacenters around the world."

https://cloud.google.com/spanner/docs/true-time-external-consistency#:~:text=FAQ-,What%20consistency%20guarantees%20does%20Cloud%20Spanner%20provide%3F,just%20those%20within%20a%20partition.

99,99958 % availability

https://research.google/pubs/pub39966/
https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45855.pdf
https://www.cockroachlabs.com/blog/spanner-vs-cockroachdb/

#### Fauna

"Why Strong Consistency Matters with Event-driven Architectures"

https://fauna.com/blog/why-strong-consistency-with-event-driven

"ACID compliance ensures consistency your app can depend on. Real-time event streaming powers rich, interactive applications."

https://fauna.com/solutions#edge-computing

"Use it with edge computing vendors such as Cloudflare, Fastly, or Azion to offer rich, low latency experiences for users everywhere."

#### MongoDB

- Kind-of Raft
- Strongly consistent (TODO refer to consistency types in 03a)
    - Consistent + Partition-Tolerant (CP)
- Pull-based consensus
- User can decide: Also supports causal consistency (https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/#causal-consistency)

PACELC: MongoDB can be classified as a PA/EC system. In the baseline case, the system guarantees reads and writes to be consistent.

TODO rephrase

"MongoDB provides linearizability and tolerates any minority of failures
through a novel consensus protocol that derives from Raft. A
major difference between our protocol and vanilla Raft is that
MongoDB deploys a unique pull-based data synchronization
model: a replica pulls new data from another replica. This
pull-based data synchronization in MongoDB can be initiated
by any replica and can happen between any two replicas, as
opposed to vanilla Raft, where new data can only be pushed
from the primary to other replicas. This flexible data transmission topology enabled by the pull-based model is strongly
desired by our users since it has an edge on performance and
monetary cost [@zhou2021fault]."

"MongoDB is strongly consistent by default: reads and writes are performed across the primary member of a replica set. Applications can optionally read from secondary replicas where the data is eventually consistent by default."

#### etcd / Kubernetes

- Raft

#### Apache Ozone

Apache Ozone is a... [@apache2022ozone]
The replication layer of Apache Ozone is built upon Apache Ratis, a library for..., []
Apaache Ratis is the library of choice for the implementation of a replicated event store presented in the next chapter.

#### Azure Cosmos DB

- Configurable
- Azure Cosmos DB offers five(!) well-defined levels. From strongest to weakest, the levels are:

- Strong
- Bounded staleness
- Session
- Consistent prefix
- Eventual
- https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels

"With Azure Cosmos DB, developers do not have to settle for extreme consistency choices (strong vs. eventual consistency). It offers 5 well-defined consistency choices - strong, bounded-staleness, session, consistent-prefix and eventual – so you can select the consistency model that is just right for your app." [@microsoft2022cosmosconsistency]

- TLA+ spec proven: https://github.com/Azure/azure-cosmos-tla
- Leslie Lamport co-authoring: https://www.youtube.com/watch?v=kYX6UrY_ooA

"Extreme aggressive SLA"

#### Other Mentionable Implementations

> TODO should summarize, not an own section per tool

- Hadoop
- Neo4j
- Couchbase (Also NewSQL)
- Apache Spark
- Apache Flink
- Apache Cassandra
    - Availability+Partition Tolerance
- Redis (https://github.com/RedisLabs/redisraft), still working on it, not production-ready
    - https://redis.com/blog/redisraft-new-strong-consistency-deployment-option/ 
    - Consistency+P
     distributed, in-memory key–value database, cache and message broker,

#### Comparison Results

Lorem ipsum...

### Popular Event Stores and Time-Series Databases

Lorem ipsum...

\todo{Evaluate performance of those and mention their consistency models}
\todo{Have a table like in the upper section}
\todo{Performance measure here or in 05 evaluation?}

#### EventStoreDB

Consensus; is it raft?
https://developers.eventstore.com/server/v21.10/cluster.html#cluster-nodes

#### TimescaleDB

Lorem ipsum...

#### InfluxDB

"The most related storage system is InfluxDB [9], an open-source time series database solution. Unlike ChronicleDB, InfluxDB is not optimized for single machine data storage but is designed from
the ground up as a distributed system. Furthermore, in its current state, it is more suited for univariate time series data and, just like other time series systems, has no support for CEP-style
queries."

https://s3.amazonaws.com/vallified/InfluxDBRaft.pdf
https://www.influxdata.com/blog/influxdb-clustering/

What is stored via consensus?
Only Metadata:

- Cluster membership
- Databases
- Retention Policies
- Users
- Continuous Queries
- Shard Metadata

What is not stored via consensus?

- The time-series data itself, the indexes and schemas


To not store the data has been introduced with a newer version, before that, all data went through raft (<= v0.9-rc20)
    - Potentially high performance
    - Global schema which meant improved user experience
    - Easier to reason about when all data in consensus
- In practice it was inefficient and complex to implement correctly.
- Approach was abandoned by 0.9.0

Planned design in 0.10.0: Distinct Raft subsystem + data-only nodes

https://www.influxdata.com/blog/multiple-data-center-replication-influxdb/

- primary-copy (?) for  Data Center Replication, Not built-in:
https://github.com/influxdata/influxdb-relay 
This project adds a basic high availability layer to InfluxDB. With the right architecture and disaster recovery processes, this achieves a highly available setup.

https://github.com/toni-moreno/syncflux

Lorem ipsum...

#### Apache IoTDB

[@wang2020iotdb]

- https://iotdb.apache.org/
- Raft (TODO only for meta or also for data?)
- Meta Group, Data Group
- https://github.com/apache/iotdb/issues/3954
- https://github.com/apache/iotdb/tree/master/consensus
    - It is heavily WIP, but it's raft consensus is built on top of Ratis, same as for our implementation
    - But they built a layer around it, to be agnostic of the protocol and framework https://github.com/apache/iotdb/pull/5939
        - https://github.com/apache/iotdb/blob/master/consensus/src/main/java/org/apache/iotdb/consensus/ConsensusFactory.java
        - One API for all protocols
    - They aim to use different protocols for different parts of the db (meta, region management, data...)

#### Prometheus

Lorem ipsum...

#### Apache Kafka

TODO PubSub etc., Differentiation to event stores

#### Other Mentionable Implementations

Lorem ipsum...

#### Use Cases and Differentiation to ChronicleDB

Lorem ipsum...

#### Comparison Results

Lorem ipsum...

\pagebreak

## Previous Implementations {#sec:previous-work}

\epigraph{If I have seen further it is by standing on the shoulders of Giants.}{--- \textup{Isaac Newton}}

In subchapter [@sec:why-replication], we discussed consistency models and several replication protocols. In this subchapter, we take a look at existing implementations of replication protocols in practice by reviewing several selected distributed databases for event-like data. We will investigate their approaches to replication, the impact of these approaches on performance and client consistency, and what we can learn from them. It is important to evaluate the impact of replication protocols in practice, as some theoretical constraints have a negligible impact on practical behavior, while the impact of some other considerations is evident in practical applications.

### Criteria for Review and Comparison

Since there are a multitude of distributed databases, we need to narrow the scope for our consideration.

We are looking for databases and similar system that store and manage event-like data, such as time series or messages. Our focus is on replication mainly for fault-tolerance and increased availability. We do not look at databases that only offer geo-replication. We also do not look at decentralized systems, such as blockchains.<!--We are looking at distributed databases that can be deployed as a cluster of replicas, including NoSQL as well as new relational databases (NewSQL), but also event messaging and storaging systems.-->

<!--
### Distributed Databases

We look at various databases and other related applications, and summarize and compare their properties in table \ref{table:systems-studied}. We will cover event stores and similar systems in section [@sec:distributed-event-stores].

Also look at https://www.researchgate.net/figure/Consistency-models-comparison_tbl1_331104869

\begin{table}[h!]
    \caption[List of systems studied in this work]{The list of systems studied in this work, and their primary dependability and consistency properties}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}l X X} 
        \toprule
        \thead{System} & \thead{Consistency Model} & \thead{Replication Protocol} \\
        \midrule
        \multicolumn{3}{c}{\thead{Key-Value Stores \& Registries}} \\
        \midrule
        LogCabin & Strong & Raft \\
        etcd (Kubernetes) & Strong & Raft \\
        Zookeeper & Strong & Primary-Copy \\
        Redis & Eventual, Strong is WIP & CRDTs, Primary-Copy, Raft is WIP \\
        \midrule
        \multicolumn{3}{c}{\thead{NoSQL \& Object Stores}} \\
        \midrule
        MongoDB & Strong & Pull-Based Consensus (Raft-alike) \\
        Apache Cassandra & User Choice (Eventual—Strong) & - \\
        \midrule
        \multicolumn{3}{c}{\thead{Process \& Service Orchestration}} \\
        \midrule
        Zeebe & Strong & Raft \\
        AAA & - & - \\
    \end{tabularx}    
    \label{table:systems-studied}
\end{table}

\todo{consistency of other orchestration engines, such as orkes?}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/CAP-dbs.pdf}
  \caption[Classification of database systems with the CAP theorem]{Classification of different database systems into the categories of the CAP theorem}
  \label{fig:cap-dbs}
\end{figure}

TODO many of these systems allow their users to decide for the consistency model and therefore for the availability-consistency (and the latency-consistency, considering the PACELC theorem) trade-off.

#### LogCabin

Written by the author of Raft... [@ongaro2015logcabin]
... this thesis' implementation for metadata and cluster management follows similar principle... see Implementation Section 

#### etcd / Kubernetes

- Raft

#### CockroachDB

- Raft
- NewSQL: Offering benefits of distributed systems with scalable OLTP while still providing ACID capability for relational databases [@pavlo2016s]

- Problem with WAL on raft log https://github.com/cockroachdb/cockroach/issues/38322

#### Camunda Zeebe

- TODO add stuff from our pages
- https://github.com/camunda/zeebe/tree/main/raft/src/main/java/io/zeebe/raft/state
- docs... etc

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
built on Googles infrastructure, therefore even with strong consistency, it is highly available and can even use real-time timestamps. This is only possible due to the fact that Google is in full control of the infrastructure. "Even though it provides such strong guarantees, Cloud Spanner enables applications to achieve performance comparable to databases that provide weaker guarantees (in return for higher performance)."

"Spanner uses the Paxos algorithm as part of its operation to shard (partition) data across up to hundreds of servers.[1] It makes heavy use of hardware-assisted clock synchronization using GPS clocks and atomic clocks to ensure global consistency.[1] TrueTime is the brand name for Google's distributed cloud infrastructure, which provides Spanner with the ability to generate monotonically increasing timestamps in datacenters around the world."

https://cloud.google.com/spanner/docs/true-time-external-consistency#:~:text=FAQ-,What%20consistency%20guarantees%20does%20Cloud%20Spanner%20provide%3F,just%20those%20within%20a%20partition.

99,99958 % availability

https://research.google/pubs/pub39966/
https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45855.pdf
https://www.cockroachlabs.com/blog/spanner-vs-cockroachdb/


#### MongoDB

Time series: https://www.mongodb.com/docs/manual/core/timeseries-collections/

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

#### Apache Ozone

Apache Ozone is a... [@apache2022ozone]
The replication layer of Apache Ozone is built upon Apache Ratis, a library for..., []
Apaache Ratis is the library of choice for the implementation of a replicated event store presented in the next chapter.

Apache Ozone is built for high-throughput, which demonstrates that the Raft consensus protocol is also suitable for data-intense applications, considering some optimizations of the original protocol.

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


#### Comparison Results

Lorem ipsum...

-->

### Distributed Event Stores and Time-Series Databases {#sec:distributed-event-stores}

We look at various event stores, time-series databases and related applications, such as pub/sub messaging systems, and summarize and compare their properties in table \ref{table:event-stores-studied}.

\begin{table}[h!]
    \caption[List of event stores and similar systems]{The list of event stores and similar systems studied in this work, and their primary dependability and consistency properties}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\small\bfseries}l Z Z} 
        \toprule
        \thead{System} & \thead{Consistency Model} & \thead{Replication Protocol} \\       
        \midrule
        \multicolumn{3}{c}{\thead{Event Stores}} \\
        \midrule
        EventStoreDB & Strong—Eventual (?) & Custom \\
        Azure Event Hubs & Strong—Eventual (per partition) & Unknown (proprietary) \\
        \midrule
        \multicolumn{3}{c}{\thead{Time-Series DBs}} \\
        \midrule
        InfluxDB & Meta: Strong. Data: Eventual & Meta: Raft. Data: Custom \\
        Apache IoTDB & Strong & Raft (Ratis) \\
        \midrule
        \multicolumn{3}{c}{\thead{Acounting Databases}} \\
        \midrule
        TigerBeetle & Strong & Viewstamped Replication \\
        \midrule
        \multicolumn{3}{c}{\thead{Pub/Sub \& Message Brokers}} \\
        \midrule
        Apache Kafka & Meta: Strong. Data: Strong—Eventual (per topic) & Meta: ZooKeeper (Now), KRaft (Proposal). Data: Primary-Copy \\
        RabbitMQ & Strong-Eventual (per queue) & Raft-based  \\
        \bottomrule
    \end{tabularx}
    \label{table:event-stores-studied}
\end{table}

#### InfluxDB

The original ChronicleDB paper mentions InfluxDB as the "most related storage system". InfluxDB is an open-source time-series database that was natively developed to operate in a distributed cluster. It is not optimized to run on a single machine or embedded in another application, such as ChronicleDB. As a time-series database, it works best with univariate time series data and does not support queries for complex event processing. It provides its own storage layout, called time-structured merge tree (TSM) (which, as the name suggests, is somehow based on _log-structured merge trees_ (LSM)). It is also suitable for long-term data storage.

InfluxDB does not provide strong consistency for data. They provide two seperate layers of consistency[^influx-cap] [@influx2022clustering]: 

[^influx-cap]: Also see their argumentation why they are neither CP or AP, according the the CAP theorem https://www.influxdata.com/blog/influxdb-clustering-design-neither-strictly-cp-or-ap/

##### Meta Nodes.

Meta nodes store metadata for cluster management with strong consistency using a distinct Raft subsystem. Meta nodes cover the following data:

- Cluster membership,
- Retention policies,
- Users,
- Continuous queries,
- Shard metadata.

##### Data Nodes. 

Data nodes store the actual payload (the time-series data itself, the indexes and schemas) seperately and with an own replication scheme that provides eventual consistency (in practice only under very high throughputs, according to an own statement), called _hinted handoff_. Therefore, the data nodes can already be operated with a low replication factor of 2 (without a quorum), while a quorum mode is also possible - but still only eventually consistent.

In the past (before version 0.10), InfluxDB also used Raft for data nodes[^influx-old] [@influx2015replication]. Since they were not able to meet throughput requirements with their implementation of Raft since "In practice it was inefficient and complex to implement correctly", they decided to remove Raft for data nodes and to introduce an eventually consistent protocol. Their latest statement to this decision reads: "InfluxDB prioritizes read and write requests over strong consistency. InfluxDB returns results when a query is executed. Any transactions that affect the queried data are processed subsequently to ensure that data is eventually consistent. Therefore, if the ingest rate is high (multiple writes per ms), query results may not include the most recent data" [@influx2022design].

[^influx-old]: The documentation on old versions with full Raft, but in alpha state, can still be found in their archives: https://archive.docs.influxdata.com/influxdb/v0.9/. 

<!--
Out-of-order:
"The vast majority of writes are for data with very recent timestamps and the data is added in time ascending order.
Pro: Adding data in time ascending order is significantly more performant
Con: Writing points with random times or with time not in ascending order is significantly less performant"
-->

##### Geo-Replication.

While they do not provide their own solution to geo-replication and disaster recovery, they point out to possible solutions using other tools like Kafka or specialized tools for InfluxDB built by third parties[^influx-third-party-geo-replication] [@influx2017datacentereplication]. 

[^influx-third-party-geo-replication]: Also see for reference https://github.com/influxdata/influxdb-relay and https://github.com/toni-moreno/syncflux.

#### Apache IoTDB

Apache IoTDB (Database for Internet of Things) is a time-series database specifically made to work in an edge-cloud setting. It can be deployed on all three layers in edge computing: embedded on edge appliance, e.g. on IoT devices, in the terminal layer, standalone on the edge layer and distributed on the cloud [@wang2020iotdb]. This is actually pretty similar to our approach for ChronicleDB. The open-source project started in 2019 and was adopted by Apache after it was accepted as an Apache Incubator project.

Following is a list of IoTDBs key features:

- Apache IoTDB is developed to work closely with existing open-source stream and batch processing tools, providing out of the box connectors for Apache Spark, Flink and Hive (Hadoop)[^iotdb-web].
- In IoTDB, data is partitioned by time splits (here, it is called _time slices_).
- It comes with a custom storage layout.
- It features a config node and a data node group, similar to InfluxDB.
- In addition, data and schemas are partitioned seperately, while the replication factor and consensus protocol for both can differ.
- Its default replication protocol is Raft.
- To move data between the edge-cloud layers, if provides an own sync protocol and service, that replicates the time-series data on a file level.

The project is still under heavy construction, especially its consensus layer. Currently, they use _Apache Ratis_ to implement the Raft protocol—to say it in advance: We have also opted for this library. At the moment, they build an aditional adapter layer around it, allowing them to be agnostic of the underlying protocol and framework. This adapter also allows to switch between embedded/standalone and cluster operation modes, without having to build and maintain 2 different architectures[^iotdb-github-consensus].

[^iotdb-web]: https://iotdb.apache.org/

[^iotdb-github-consensus]: Cf. the consensus module/adapter at https://github.com/apache/iotdb/tree/master/consensus and this recently merged pull request to include a multi-leader protocol (similar to ROWA, cf. subsection [@sec:rowa]) behind this adapter https://github.com/apache/iotdb/pull/5939 

#### EventStoreDB

EventStoreDB is, as the name suggests, an event store, but not a multi-purpose one. The authors of EventStoreDB call it "The stream database built for Event Sourcing". Event sourcing is in fact its main use case: it is built to store consecutive changes of a state as separate events. While sensor data could also be tracked with EventStoreDB, it is not its main purpose, and it only serves low throughput compared to ChronicleDB and other systems we investigated here: over 15.000 writes/s according to their website [@eventstoredb2022performance].

EventStoreDB can be operated in different modes. We haven't found the actual protocol and consistency details and can only make assumptions about the consistency of a EventStoreDB cluster by looking at its various operation modes and the guarantees given in each mode. They provide guarantees on writes, since by default, every write to the cluster needs to be acknowledged by all cluster members, not just a quorum. While this condition can be changed by users, the documentation strongly advises against doing so [@eventstoredb2022cluster]. They also provide roles for nodes that are similar to consensus protocols: a node can either be a leader, a follower, or a read-only replica (that can be queried, but does not partake in a quorum to acknowledge a write). Per default, the cluster runs in a single strong leader mode, which in combination with a quorum should guarantee strong consistency. They also allow users to opt out of strong leader guarantees, so together with lower write acknowledgement requirements, this may result in eventual consistency.

<!--
#### TimescaleDB

A time series DBMS optimized for fast ingest and complex queries, based on PostgreSQL...

No custom storage layout
-->

#### TigerBeetle

TigerBeetle is an append-only accounting database with strict serializability [@coil2022tigerbeetlegithub]. Since it is used in financial use cases with high business criticality, strong consistency is a must. It uses viewstamped replication under the hood, which belongs to the class of state machine replication protocols (see subsection [@sec:viewstamped]). On their website, they claim that they provide a high throughput of approximately 1 million journal entries per second, while providing "extreme" fault-tolerance [@coil2022tigerbeetle]. They make this a clear unique selling point, since their website is called "TigerBeetle - A Million Transactions Per Second".

The database itself is available open-source, written in the language `Zig`, while they offer a paid premium service.

#### QuestDB

QuestDB is a standalone time-series database. The authors claim that "QuestDB is the fastest open source time series database" [@quest2022db]. This claim is underpinned by a benchmark suite that is public and can be run by developers themselves, and their benchmark is impressive: on the same setup, QuestDB shows a peak throughput of almost 1 mio rows/s, while InfluxDB only comes up with around 330k rows/s [@quest2022benchmark].

QuestDB is currently not a distributed database. While they provide a cloud offering, this is not fault-tolerant. Based on their roadmap, they aim to support distributed reads and writes in upcoming releases[^quest-db], including handling of out-of-order events.

[^quest-db]: Cf. https://github.com/orgs/questdb/projects/1/views/5 for reads and https://github.com/questdb/roadmap/issues/12 for writes

#### Apache Kafka {#sec:kafka}

Apache Kafka is a distributed event streaming platform that can be used in a _publisher-subscriber_ (pub/sub) scheme. The pub/sub scheme describes systems that allow publishers to produce a single message or event that is consumes by multiple subscribers [@jacobsen2009pubsub]. Kafka was originally developed at LinkedIn to track user interaction events on their social media platform [@auradkar2012data], and quickly became something of a de facto standard for a variety of industries and companies, including manufacturing and finance. Nowaydays, it supports not only pub/sub message distribution, but also resilient stream processing. Kafka can also be used as a permanent storage for streams, served from a fault-tolerant high-availability cluster.

At the heart of Kafka is a replicated transaction log that manages _topics_. A topic in Kafka is similar to what we call an event stream in ChronicleDB, or what is generally called _channel_ in event processing [@etzion2009eventchannel]. A topic is a domain of interest that a consumer subscribes to, to receive all messages regarding this specific topic. Topics are partitioned and distributed across nodes, called _brokers_. Each partition is an append only log with strong insertion ordering guarantees. Other than streams, a topic can be configured to operate in multiple ways, such as in a _exactly-once_ mode, which guarantees that a message is always delivered, and exactly once, to one or more consumers, even in case of producer or consumer failures. Kafka also allows for atomic transactions across multiple topics. 

To allow for such guarantees, the replication layer of Kafka provides multiple levels of consistency for the user to select from [@wang2015building]. The meta (cluster and topic information) and data layer in Kafka are distributed with different consistency  guarantees and replication protocols. Data itself not replicated using consensus, but with some approach similar to primary-copy log replication. Users can decide on the replication and the acknowledgement level per stream. A quorum is not required. Depending on the acknowledgement level, the protocol uses a high-water mark to identify the portion of the log that is already acknowledged [@kafka2022kip101]. Since Kafka does not differentiate in ordering between insertion and event time, it can operate directly on the topic streams.

Metadata is distributed with consensus, thus providing strong consistency. Currently, the metadata replication layer and the leader election of Kafka is built on Apache ZooKeeper [@hunt2010zookeeper]. The metadata also includes crucial information for read consistency, such as the topic _offset_, which marks the last consumed event of the topics stream. We introduced ZooKeeper and the ZooKeeper Atomic Broadcast Protocol in subsection [@sec:zookeeper].

##### KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum (Raft)

Since ZooKeeper is an external service and not embedded natively in the distributed data storage system it replicates, it comes with some caveats, especially regarding performance. Therefore, the Apache Kafka team dediced to discontinue the usage of ZooKeeper and to implement their own replication layer for metadata, embedded in Kafka. You can find the motivation to do so in their often-cited _Kafka Improvement Proposal_ (KIP) 500 [@kafka2022kip500]. This replication layer is based on Raft and called _KRaft_, where the _K_ stands for Kafka. This is still under development and therefore currently not recommended to be used in production.

#### RabbitMQ

RabbitMQ is a _message broker_ built on the _AMQP_ protocol. It provides similarities to the _pub/sub_ scheme as in Apache Kafka, but it works quite differently. It's main use case is messaging, and it provides powerful routing capabilities for messages that goes beyond subscribing to a topic. RabbitMQ is commonly used in telecom industries thanks to this routing capabilities and because it built upon the _Open Telecom Platform_ (OTP) which is a core part of the open-source distribution of `Erlang`.

Most of the distribution and replication design of RabbitMQ is based on the OTP. But recently, the authors introduced Raft for _quorum queues_ [@rabbitmq2021quorum]. This allows RabbitMQ users to opt-in to strong consistency on a queue level. A queue is the equivalent to a channel in event processing [@etzion2009eventchannel] or an event store stream. On the RabbitMQ documentation, they highlight that quorum queues should be used in critical cases where fault tolerance and data safety is more important then latency and advanced queue features—effectively saying that a CAP trade-off needs to be made.

<!--

#### Fauna

"Why Strong Consistency Matters with Event-driven Architectures"

https://fauna.com/blog/why-strong-consistency-with-event-driven

"ACID compliance ensures consistency your app can depend on. Real-time event streaming powers rich, interactive applications."

https://fauna.com/solutions#edge-computing

"Use it with edge computing vendors such as Cloudflare, Fastly, or Azion to offer rich, low latency experiences for users everywhere."

-->

#### Azure Event Hubs

Azure Event Hubs is a _Platform as a Service_ (PaaS) for event streaming. It provides full integration into the Kafka ecosystem, and allows to natively connect realtime stream processing and batch processing systems of the Azure service portfolio, as well as event sinks for long-term storage. Microsoft claims that it can ingest millions of events per second [@microsoft2022eventhubs], since it allows to scale linearly with stream sharding.

Event Hubs allow users to choose the consistency level on a per-partition (shard) basis. Users can choose between eventual and strong consistency. With strong consistency, the ordering of the events are guaranteed. Across partitions, they do not provide consistency, since each partition is covered by its own replication protocol instance [@microsoft2022eventhubsconsistency]. They also do not provide an query consistency layer that restores consistency on reads, so users are adviced to either use single-partition streams or to handle this on the event consumer side.

#### Comparison Results

Our analysis shows that high throughput can be guaranteed even with strong consistency (e.g., TigerBeetle, IoTDB). Most vendors separate data and metadata when it comes to replication. Some vendors point out the tradeoffs and have chosen not to provide strong consistency at the data level, while others let the user decide the level of consistency. What most have in common is that they provide strong consistency for metadata (cluster information, stream watermarks, and other information that is not pure payload data).

\pagebreak

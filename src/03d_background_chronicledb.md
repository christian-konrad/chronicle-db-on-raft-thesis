## ChronicleDB, a High-Performance Event Store {#sec:chronicle-db}

\todo{Brief introduction}

In this chapter, we introduce the event store that this work is about: ChronicleDB, a high-performance event store [@seidemann2019chronicledb]. Throughout subsequent chapters of this work, we will implement a replication layer that allows us to run ChronicleDB as a distributed event store in both the cloud and on the edge, while maintaining the excellent capabilities of standalone operation or embedding in edge devices such as sensors or mobile devices. In this introduction, we will focus on the core aspects of ChronicleDB that are relevant for operation in a replicated environment. For further details and the entire specification, we refer to the work of Seidemann et al. [@seidemann2019chronicledb], Glombiewski et al. [@glombiewski2020designing], and the dissertation by Körber [@koerber2021accelerating].

"write-optimized data stores
have emerged recently. However, the drawbacks of these systems are a fair query performance and the lack
of suitable instant recovery mechanisms in case of system failures.
In this article, we present ChronicleDB, a novel database system "

"ChronicleDB, a novel database system with a storage layout tailored for high
write performance under fluctuating data rates and powerful indexing capabilities to support a variety of
queries. In addition, ChronicleDB offers low-cost fault tolerance and instant recovery within milliseconds.
Unlike previous work, ChronicleDB is designed either as a serverless library to be tightly integrated in an
application or as a standalone database server. Our results of an experimental evaluation with real and synthetic data reveal that ChronicleDB clearly outperforms competing systems with respect to both write and
query performance."

--> Fault-tolerance / recovery as far as possible for non-replicated 

- Currently Only Embedded as Library or Standalone, not suitable for Distributed Systems: No horizontal scalability
- TODO reference what Bernd from Camunda has written on this (Embedded vs. Horizontal Scalability)

### Use Cases

\todo{Rephrase and move parts to Architecture}

"The rapidly growing Internet of Things (IoT) is reinforcing the new challenge for present-day data
processing. Sensor-equipped devices are becoming omnipresent in companies, smart buildings,
airplanes, and self-driving cars. Furthermore, scientific observational (sensor) data is crucial in
various research, spanning from climate research over animal tracking to IT security monitoring."

"All these applications rely on low-latency processing of events attached with one or multiple temporal attributes."


"not only the analysis of such events, but also
their pure ingestion is becoming a big challenge. On-the-fly event stream processing is not always
the outright solution. Many fields of applications require maintaining the collected historical data
as time series for the long term, e.g., for further temporal analysis or provenance reasons."

"For example, in the field of IT security, historical data is crucial to reproduce critical security incidents
and to derive new security patterns. This requires writing various sensor measurements arriving at
high and fluctuating speed to persistent storage. **Data loss due to system failures or system overload is generally not acceptable**, as these data sets are of utmost importance in operational applications"


"Currently, there is a lack of systems supporting the above-described workload scenario (write-intensive, ad hoc temporal queries, and fault-tolerance). Standard database systems are not designed for supporting such write-intensive workloads. Their separation of data and transaction logs generally incurs high overheads."

"To overcome these deficiencies, particularly the poor write performance, log-only systems have emerged where the log is the only data repository [18, 43, 44]. Our system ChronicleDB also keeps its data in a log, but it differs from previous approaches by its centralized design for high volume event streams. Because there are often only modest changes in event streams, ChronicleDB exploits the great potential of their lossless compression to boost write and read performance beyond that of previous log-only approaches."

"design of a novel storage layout to achieve fault tolerance and near-instant recovery within milliseconds in case of a system failure"

- Types of Requests....

"ChronicleDB can either be plugged into applications
as a library or run as a centralized system without the necessity to use the common distributed storage stack"

"The domain of ChronicleDB partly relates to different types of storage systems, including data warehouse, event log processing, as well as temporal database systems."

"ChronicleDB should be as economical as possible with regard to storage utilization to store data for the long term, i.e., months or years. Therefore, we aimed at a centralized storage system for cheap disks running as an embedded storage solution within a system"

### Requirements and Architecture

\epigraph{\hfill The log is the database.}{}

In ChronicleDB, the major design principle is: _the log is the database_.

"We avoid additional logging, as the considered append-only scenario does not cause costly random I/Os, and therefore it does not incur buffering strategies with no-force writes. Only in case of out-of-order arrival of events do we have to deviate from this paradigm, as we want to keep the data ordered with respect to application time"

\begin{figure}[h]
  \centering
  \includegraphics[width=0.45\textwidth]{images/chronicle-architecture.pdf}
  \caption{Overview of the ChronicleDB architecture}
  \label{fig:chronicle-architecture}
\end{figure}

#### Events and Streams

ChronicleDB aims at supporting temporal-relational events, which consist of a timestamp t and
several non-temporal, primitive attributes ai . Sequences of events (streams) can be considered as
multi-variate time series, but with non-equidistant timestamps. Timestamps can either refer to
system time (when the event occurred at the system) or application time (when the event occurred
in the application). The latter is more meaningful to temporal queries on the application level, and
thus, our goal is to maintain a physical order on application time

"Generally, we assume events to arrive chronologically. Events are inserted into the system once and are (possibly) deleted once. In the meantime, there are no updates on an event. They form an immutable append-only log"

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/chronicle-queue-topology.pdf}
  \caption[Example of a ChronicleDB queue topology for 6 event streams]{Illustrative example of a ChronicleDB queue topology for 6 event streams}
  \label{fig:chronicle-queue-topology}
\end{figure}

#### Queries

The most important types of queries the system has to support are _time travel queries_ and _temporal aggregation queries_. Time travel queries allow requests for specific points and ranges in time,
e.g., all ssh login attempts within the last hour. Temporal aggregation queries give a comprehensive
overview of the data, e.g., the average number of ssh logins for each day of the week during the last
three months. In addition, the system should efficiently support queries on non-temporal attributes,
e.g., all ssh logins within the last day from a certain IP range, and also be capable to provide efficient,
index-based replay of common event-processing queries like pattern matching

#### Storage Engine

"The storage engine of ChronicleDB logically consists of three components: event queues, workers, and disks. Event queues are memory-based FIFO queues, decoupling event producers, and
ChronicleDB. We maintain one event queue per-event stream, each one processed by exactly one
worker thread. Worker threads extract events from the in-memory queues, handle serialization
and compression, and finally store them on the configured disk. Figure 2 depicts an example configuration of ChronicleDB. The system processes six event streams and hence manages six event
queues. These queues are assigned to three worker threads, which in turn store the event data on
two different disks."

\begin{figure}[h]
  \centering
  \includegraphics[width=0.45\textwidth]{images/chronicle-macro-blocks.pdf}
  \caption{Layout of macro blocks in the ChronicleDB storage}
  \label{fig:chronicle-macro-blocks}
\end{figure}

### TAB+ Tree Index

- Time splits
- Time complexity: For append only and random write (= OOO merge)
- TODO argument why a CRDT won't work because merge is too expensive 

#### Time-Splitting

"ChronicleDB is a hybrid between OLTP and OLAP databases.  In terms of data ingestion, ChronicleDB is like a traditional OLTP system, but queries to ChronicleDB are similar to OLAP queries. Aggregation queries on (historical) data are essential in OLAP systems and commonly address predefined time ranges, like the sales within the past week or month"

#### Fault-Tolerance

"design of a novel storage layout to achieve fault tolerance and near-instant recovery within milliseconds in case of a system failure"

#### Aggregates

"In addition to lightweight temporal indexing, ChronicleDB offers adaptive secondary indexing support to significantly speed up non-temporal queries on its log."

"ChronicleDB’s lightweight indexing approach maintains small sets of
aggregates per attribute (min,max,count,sum) on different levels of its primary index to summarize the stored event data for different time granularities."

"Moerkotte introduced Small Materialized Aggregates (SMA) [33]
to speed up query processing in data warehouses. SMAs store a sequence of aggregations in a separate file on disk, whereby each aggregate value corresponds to a sequence of consecutive tuples
(e.g., a page). In contrast, ChronicleDB stores these aggregates within its primary index and thus
omits random writes introduced by separate index files. Further, we manage aggregates hierarchically and this way provide summarizations of different granularities instead of only a single one."

"ChronicleDB keeps all secondary information within the primary index, and thus omits random writes during index creation. In addition, queries on multiple attributes do not need to access multiple indexes. ChronicleDB computes these aggregates incrementally as new events arrive."

"To efficiently support queries on non-temporal attributes without high temporal correlation, ChronicleDB also provides secondary indexes"

#### Support for Out-Of-Order Events {#sec:chronicle-out-of-order}

Generally, we assume events to arrive chronologically. However, we also want to support occasional out-of-order insertions, as they typically occur in event-based applications [15]. They can happen, e.g., if the clocks of the sensors are skewed or simply due to
communication problems.

"To support out-of-order arrival of events, we developed a hybrid logging approach between our log storage and traditional logging."

"extended load scheduling approach ensuring high ingestion rates under fluctuating data rates and high fractions of out-of-order arrivals."

"We explicitly handle out-of-order events by preserving a certain amount of spare
space inside the pages written to disk. This way, late events can be absorbed without splitting existing nodes. However, if the spare space is exhausted or the out-of-order rate is to high, the number
of random I/Os increases drastically and consequently the ingestion performance suffers. If this
situation persists, then the input queues may run full, resulting in back pressure to the producer at least and data loss due to dropped events at worst."

### Segmented Block Storage

### Lambda Engine

TODO describe what we have written in system design again here

<!--

In general, we distinguish between real-time and non-real-time applications when it comes to stream and event processing. Commonly applied system architecture approaches provide both real-time and non-real-time processing to users and clients. Two popular architectures that provide this approach are the _Kappa architecture_ and the _Lambda architecture_. Both architectures allow for real-time and historical analytics in a single setup. The Lambda architecture requires that event processing reflects the latest state in both the batch and real-time streaming layers (called the _speed layer_), hence requiring consistency. At the heart of the Lambda architecture is an append-only data source—such as ChronicleDB, since it allows for both continous and ad-hoc queries.

What makes ChronicleDB special here is that it can act both as a stream provider, data lake and warehouse. It unifies both OLTP and OLAP properties, making it possible to replace multiple lined up systems (such as Apache Kafka with a Hadoop or Cassandra instance for batch processing) in a Lambda architecture with a ChronicleDB cluster

-->

More benefits... [@glombiewski2020designing]

### Relation to Event Stream Processing

ChronicleDB keeps all secondary
information within the primary index, and thus omits random writes during index creation. In
addition, queries on multiple attributes do not need to access multiple indexes. ChronicleDB computes these aggregates incrementally as new events arrive. The applied techniques are closely
related to sliding window aggregation in data stream processing systems [28, 29, 39]


Event Stream Processing. ChronicleDB is designed to run side-by-side with event stream
processing systems such as Esper1 or Apache Flink [19]. It provides the ability to re-run continuous queries on historical event data and boost their performance via indexes. In this work, we focus
on two important operations: sliding window aggregation and pattern matching.

\pagebreak

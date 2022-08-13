## ChronicleDB, a High-Performance Event Store {#sec:chronicle-db}

In this chapter, we introduce the particular event store that this work focuses on: _ChronicleDB, a high-performance event store_ [@seidemann2019chronicledb]. Throughout subsequent chapters of this work, we will implement a replication layer that allows us to run ChronicleDB as a distributed event store in both the cloud and on the edge, while maintaining the capabilities of standalone operation and embedding in edge devices such as sensors or mobile devices. In this introduction, we will not explain ChronicleDB in detail, but focus on the core aspects of ChronicleDB that are relevant to operating in a replicated environment. For further details and the entire specification, we refer to the work of Seidemann et al. [@seidemann2019chronicledb] and Glombiewski et al. [@glombiewski2020designing], and also to the dissertation of Körber on online and offline stream processing [@koerber2021accelerating].

ChronicleDB is an event store optimized for a high write performance under fluctuating data rates, but it also comes with powerful indexing capabilities to support a variety of continuous and ad-hoc queries. Experimental evaluation has shown that ChronicleDB outperforms competitors in terms of read and write performance [@seidemann2019chronicledb]. Before this work, ChronicleDB was designed to operate in two modes: either as a serverless library to be embedded in another application or as a standalone database server. It also provides fault-tolerance and fast failover behavior in embedded and standalone operation, which comes in handy as it increases overall availability and reliability also in distributed operations.

<!-- TODO reference what Bernd Ruecker has written on this (Embedded vs. Horizontal Scalability) -->

### Use Cases

ChronicleDB serves a broad range of use cases, due to its support for various query types. It supports both continuous and ad-hoc queries, allowing it to be used in both real-time and batch processing systems.

ChronicleDB is therefore suited for all applications that rely on low-latency processing of events that may occur in random, but very short intervals. We listed these use cases in section [@sec:event-store-use-cases].

When discussing these use cases, it is important to seperate between those with a demand for strict ordering guarantees, requiring strongly consistent streams at read time, and those that are ok with lower safety guarantees in exchange for lower latency and higher throughput. It is also important to look both at the event producer and consumer side in such an argument. We conduct this argument in detail in section [@sec:consistency-choice].

##### Relationship to DSMS.

ChronicleDB may not be considered a DSMS in the classic sense, but in an extended sense: although the stream data is stored first on disk, it is also immediately available for continuous queries, and running aggregates are stored and continuously updated. In addition, ChronicleDB extends the requirements for a DSMS, as on-the-fly processing of event streams without persisting the streams is not always a sufficient solution. In many use cases, it is necessary to keep the collected data permanently, e.g. for future ad-hoc queries, to compare real-time processing results with historical data, and to detect trends and changes.

For instance, in the realm of IT security, historical data is crucial since critical security incidents must be reproduced to derive new security patterns. For this purpose, various streams arriving at a high and fluctuating rate must be stored in a persistent storage [@seidemann2019chronicledb].

ChronicleDB was designed not only for performance, but also to support this cases, since at the moment, there is a lack of systems supporting high throughput writes, continuous queries, ad-hoc temporal queries, and fault-tolerance at the same time.

### Relationship to Event Stream Processing.

ChronicleDB is designed to run next to event stream processing systems. ChronicleDB offers outstanding support for sliding window aggregation and pattern matching queries. Thanks to its lightweight indexing, it can compute these aggregates incrementally as new events arrive. Secondary information is kept within the primary index, which allows to avoid random writes during index creation. This also removes the need to access multiple indexes when querying multiple attributes, speeding up both real-time and ad-hoc queries. This makes ChronicleDB a great companion to event stream processing systems. As mentioned before, ChronicleDB does not only support real-time queries, but it also provides the ability to replay continuous queries again on historical event data.

### Requirements and Architecture

\epigraph{\hfill The log is the database.}{--- \textup{Seidemann et al.}}

In ChronicleDB, the major design principle is: _the log is the database_. ChronicleDB leverages the properties of append-only log structures and in addition, it tries to avoid additional logging, as the considered append-only scenario does not cause costly random I/Os, and therefore it does not incur buffering strategies with no-force writes. There is one exception in case of out-of-order arrival of events: these are written into a write-ahead log first, since events should be kept ordered by event time.

In figure \ref{fig:chronicle-architecture}, we have outlined the architecture of ChronicleDB based on the descriptions in the original paper. ChronicleDB consists of a query engine to serve both continuous queries and ad-hoc queries, either via or a Java API, HTTP API or ChronicleDB-flavored SQL. The same accounts to writes. Reads and writes are served via load scheduler (illustrated in figure \ref{fig:chronicle-queue-topology}) from and to the storage engine. The storage engine provides powerful indexing capabilities (section [@sec:chronicledb-index]), a block storage layout with very fast lossless compression (section [@sec:chronicledb-storage]), and event serialization.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.45\textwidth]{images/chronicle-architecture.pdf}
  \caption{Overview of the ChronicleDB architecture}
  \label{fig:chronicle-architecture}
\end{figure}

#### Events and Streams

ChronicleDB supports temporal-relational events (see definition \ref{def:event}). Event streams in ChronicleDB are append-only log structures similar to multi-variate time series in time-series databases, but with non-equidistant timestamps. Timestamps can either refer to insertion time or event time (see subsection [@sec:event-time]). Since event time is more meaningful in queries for most consumers, ChronicleDB tries to maintain a physical order on application time [@seidemann2019chronicledb]. Once an event is inserted into a stream, it is immutable.

Inserting events in standalone ChronicleDB happens with multi-threading support to increase write throughput when multiple streams are served at the same time. Multiple job workers are instantiated to care of writing to the right stream and the right disk on the operating machine. This is illustrated in figure \ref{fig:chronicle-queue-topology}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/chronicle-queue-topology.pdf}
  \caption[Example of a ChronicleDB queue topology for 6 event streams]{Illustrative example of a ChronicleDB queue topology for 6 event streams}
  \label{fig:chronicle-queue-topology}
\end{figure}

#### Queries

Besides continuous queries, ChronicleDB has special support for _time travel queries_ and _temporal aggregation queries_. Time travel queries allow to fetch events for specific ranges in time, e.g. all orders that have been fulfilled in an e-commerce system within the last hour. It allows consumers to derive a snapshot of the state of the depicted real-world at any point in time. Temporal aggregation queries give a comprehensive overview of the data, e.g., the average number of fulfilled orders for each day of the week during the last three months. Moreover, ChronicleDB supports queries on _non-temporal attributes_, e.g., all fulfilled orders within the last day for a specific product. It also supports efficient replay of common complex real-time event processing queries, such as pattern matching, at any time.

You can read more on ChronicleDBs query capabilities, such as temporal pattern matching, sliding window aggregation, and replay of continuous queries, in the original paper [@seidemann2019chronicledb].

#### Storage Engine {#sec:chronicledb-storage}

To decouple event producers from indexing and disk writing, memory-based FIFO event queues are placed in between producers and disks. Job workers continuously fetch entries from these queues and write them to disk, while adding the index entries. This is shown in figure \ref{fig:chronicle-queue-topology}. This also allows us to run ChronicleDB efficiently in a distributed setting, since we do not need to wait for entries to be persisted on disk before acknowledging them to the replication layer.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.45\textwidth]{images/chronicle-macro-blocks.pdf}
  \caption{Layout of macro blocks in the ChronicleDB storage}
  \label{fig:chronicle-macro-blocks}
\end{figure}

The storage layout of ChronicleDB is partitioned into macro blocks. A macro block consists of C-blocks (compressed block), which in turn consist of compressed L-blocks. L-blocks (logical blocks) are of the size of a physical disk block. Macro blocks are of a fixed size with a variable amount of C-blocks and provide the smallest granularity for physical writes to disk. This setup allows for improved fail-over behavior. We refer to the original paper to learn more about ChronicleDB's compression and other storage layout features, as this is beyond the scope of this work [@seidemann2019chronicledb].

The macro blocks also contain a _spare space_ that reserves space for the insertion of out-of-order events. While the macro block is generally dense, this spare space allows for efficient insertion of out-of-order events since it helps to avoid costly block overflows.

The overall block design supports very economic storage utilization that allows to store data for the long term, while it allows for high-throughput writes at the same time. ChronicleDB runs efficienlty on centralized storage system even with cheap disks, such as in embedded systems like sensors and IoT devices.

### $\textrm{TAB}^+$ Tree Index {#sec:chronicledb-index}

The index of Chronicle is built on a _temporal aggregated $\textrm{B}^+$_ ($\textrm{TAB}^+$) tree. The $\textrm{TAB}^+$ tree is a novel approach, based on a $\textrm{B}^+$ tree that uses the timestamp of an event as its key. This approach delivers lightweight indexing that also allows small sets of aggregates to be maintained at the attribute level, such as `min`, `max`, `count` and `sum`. The aggregates are calculated and stored on the various levels of the tree, allowing to update the aggregates on the go with each arriving event. As mentioned before, secondary data is kept within the primary index, allowing to run faster queries and omitting further random writes during index creation. The index leverages the property of _temporal correlation_, which describes the observation that attributes of events occurring within a small time interval are often very similar. We illustrated the $\textrm{TAB}^+$ tree in figure \ref{fig:tabplus}. For a detailed explanation of ChronicleDB's indexing approach, including how the tree is built, how it refers to the storage layout, how it allows for fast crash recovery, how secondary indexes works and more, we refer here to the original paper [@seidemann2019chronicledb].

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/tabplus.pdf}
  \caption[The $\textrm{TAB}^+$ index layout]{The $\textrm{TAB}^+$ index layout. The nodes and leaves of the trees are connected in both directions, supporting faster queries. Each node contains pointers to its children, the minimum and maximum timestamp of the events in the branch as the index keys, the number of events per branch, and the basic aggregates per property defined in the event schema.}
  \label{fig:tabplus}
\end{figure}

#### Time-Splitting {#sec:time-splitting}

ChronicleDB allows users to align the organization of the data to their most common query patterns by introducing _regular time splits_. On a per-stream level, the user can decide the maximum period of time that one time split should cover. Once a time split reaches this size, the split is closed and a new $\textrm{TAB}^+$ tree instance is created for the subsequent current split in a seperate file. This makes historical aggregation queries on predefined time ranges more efficient, such as the total sum of fulfilled orders on a weekly basis, but it also increases write performance.

In rare cases, historic event data should be deleted (e.g., to comply with GDPR). ChronicleDB supports the removal of such events at the granularity of time splits.

### Summary

The ChronicleDB event store is a novel approach that allows for efficient online stream processing, as well as long-term persistence of events with high throughput that outperforms other temporal database systems, while supporting ad-hoc queries and reliable and fast replay of continuous queries. It runs embedded in other applications on low-cost hardware, such as IoT sensors, as well as a standalone server. Hence, it aims to serve a broad range of use cases, which me must take into account when designing a replication layer for ChronicleDB in chapter [@sec:implementation]. 

\pagebreak

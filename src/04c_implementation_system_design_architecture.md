## System Design and Architecture {#sec:system-design}

In this section, we present the design of the system, justifying the choices for dependability properties, consistency, replication protocol, libraries used, and implementation steps throughout. We also show what trade-offs we accept and when, and what this means for the final solution. We will start with a theoretical discussion, outlining the system design, and continue with a more detailed presentation. Finally, we present the implementation of our prototype, which we use to perform the benchmarking and examine the overall impact of the chosen protocol on ChronicleDB.

<!--
Developed architecture / system design / implementation: 1/3
• start with a theoretical approach
• describe the developed system/algorithm/method from a high-level point of view
• go ahead in presenting your developments in more detail
-->

<!--
- This section is about the solution design and implementation
- First list high-level requirements (strong consistency, availability, fault-tolerance...) just reference the previous chapters

- Then detailed requirements (how the user interacts with it, requirements to the API, infra and tech requirements)
-->

<!--

- We could model delete operations differently, so they could also be monotonic (= everything is an insert), by just logging them. The tree would then consist of both addEvent and deleteEvent items, and just by observing its final state we know which events are actually there. (It seems like this may does not make sense in our use cases)
- Also see https://www.slideshare.net/SusanneBraun2/keeping-calm-konsistenz-in-verteilten-systemen-leichtgemacht
- Also describe the trivial, derived monotonic aggregates thing
- TODO reference these in the conclusion. We won't build an eventual consistent prototype. But it could make sense. It can be built without any coordination mechanism between nodes if we agree for them to only use monotonic operations.
- the ohlc of a stock in a certain timeframe is a trivial derived aggregate that is suitable for our example
- insertEvent is monotonic. Aggregates are also trivial, derived monotonic aggregates -> can use eventual consistency
- but addStream, cluster management, metadata etc. must be strong consistent! (like InfluxDB did it) -> this is not monotonic (see CALM)

TODO describe that createSecondaryIndex and creation of "materialized user-defined aggregates" is also an operation in the raft log, but we won't focus on it yet - "ChronicleDB computes these aggregates incrementally as new events arrive" - so we don't care

-->

### Challenges

\epigraph{\hfill Problems are not stop signs, they are guidelines.}{--- \textup{Robert H. Schuller}}

The biggest challenge in this work was to find an appropriate trade-off between latency, availability and consistency, and therefore to find an appropriate consistency model and based on this, finding a replication protocol that satisfies the traded-off requirements. In addition, we tried to find proper ways to soften the consequences of the compromises. The trade-offs could only be made after a thorough investigation of the potentially targeted use cases of a distributed ChronicleDB (subsection [@sec:use-cases-distributed-event-store]) and a general discussion of consistency in event stores (subsection [@sec:consistency-choice]). In addition, handling of out-of-order events is challenging, especially in the face of consistency. It violates consistency, but not from the point of view of the event store, but rather from the point of view of client applications that expect a reliable stream of events. Some replication protocols require a write-ahead-log (WAL), which violates the "The log is the database" philosophy of ChronicleDB.

Throughout the next sections, we will show how we tackled these challenges, and in chapter [@sec:conclusion], we will show how these could be further mitigated in the future. But first, we limit the scope of this work to the part of the challenges that we tried to solve in the following subsection.

### Limitations

Due to the complexity of this topic, the implementation in this work is limited to a few well-justified considerations.

\paragraph{Geo-Replication.}

We limit our work to intra-cluster replication. See the conclusion for some hints for further investigations.

\paragraph{Byzantine-Fault Tolerance.}

We haven't implemented or designed a byzantine-fault tolerant version of ChronicleDB, as this work is limited to a problem scope where byzantine faults are rare (but not yet excluded).

\paragraph{Transaction Handling.}

So far, ChronicleDB does not support transactions, which is actually rare for event stores in general. This is good for us, because we don't need to think about linearizing transactions and making them atomic, which also increases the latency of such a system tremendously. If transactions are ever needed, especially across partitions, we refer to subsections [@sec:partitioning] and [@sec:coordination-free-replication].

\paragraph{Log Optimization.}

I/O on the Raft log can be costly, as all operations must be written to the log before executed on the event store replicas. Without buffering, the log looks almost similar to the actual event stream persisted in the store. In addition, it requires non-trivial state management such as allocation and pruning to prevent unbounded growth.

The log implementation in this work is far from being optimized. Even more, we violate the "The log is the database" philosophy of ChronicleDB. There are approaches for log-less state machine replication that can lead back to this philosophy [@skrzypzcak2020towards]. But, we limit our work on the replication protocol itself, not on the implementation details of the log. In chapter [@sec:conclusion], we list possible improvements on the log.

\paragraph{Log Compaction.}

We haven't implemented a snapshotting mechanism so far. Currently, when the system is rebooted, the whole log is replayed and the $\textrm{TAB}^+$ index rebuilt, which is very expensive. While snapshotting a simple key-value store is simple (which previous literature was limited to), snapshotting the event store by creating clones of it to the current point in time is too expensive, as the store is already written to disk of the replicas. We considered that it is sufficient to reduce the snapshot to a pointer that points to the latest committed event in the store, and changing the state transfer implementation of Ratis to one that transfers the $\textrm{TAB}^+$ up to this pointer. But this method may be limited to an append-only event store—we haven't considered a solution for out-of-order entries yet, nor are we sure that this requires expectional handling at all.

\paragraph{Distributed Queries.}

The following implementation in this work focuses on write replication. To query an event stream with shards on different nodes, distributed queries must be performed, which is an entirely separate, complex area of interest that exceeds the scope of this work. We refer here to the literature of distributed queries and implementations in popular distributed database systems [yu1984distributed; kossmann2000state; azhir2022join]. In this work, we have not yet implemented the query interface, even for single-shard queries.

\paragraph{Multi-Threaded Event Queue Workers and Partitioning.}

We don't know how the decoupling of local event queue threads by introducing partitioning affects the possible performance that can be achieved with multiple streams through ChronicleDBs own event queue and worker architecture, as with partitioning, the relation is no longer one-queue-per-stream, but one-partition-per-stream.

\paragraph{Production-Ready Implementation.}

The implementation made in this work is far from production-ready. I suits as a demo and to evaluate the impact and trade-offs of the chosen replication protocol and our mitigation strategies. But with the guidance and reasoning from this chapter and the possible future work in chapter [@sec:conclusion], we are very confident that it is possible to move this implementation into a production-ready system, ready to be deployed in a distributed setting, or even on the edge.

\todo{Reference the repo}

### Dependability Requirements

The unique features of event stores, compared to other database types are:

- An append-only data structure allows for extremely high throughput rates.
- To be efficient, the data structure requires linearized ordering due to the ordered nature of events.
- Not only writes, but also queries are fast thanks to time-based indexes.
- There is no concept of transactions, every single event write is atomic.
- Only a small fraction of the stored data blocks are written to, while the majority of the blocks are read-only.

In addition to this, ChronicleDB brings with it other special features:

- Aggregates can be derived in-place in the index really fast thanks to small materialized aggregates (SMA).
- Querying speed is extremely fast compared to other event stores and time series databases [@seidemann2019chronicledb].
- ChronicleDB allows for out-of-order entries and handles them efficiently as long as they do not succeed a certain threshold.
- ChronicleDB allows for fast fail-overs, even in standalone mode.
- Event schemas are fixed and are not allowed to evolve at the moment.

<!-- TODO mention use cases beforehand? -->

Based on this properties and if we imagine an ideal solution, we can raise the following requirements to a distributed event store:

- **Availability**: The event store is available for writes and queries all the time.
- **Fault-Tolerance**: The event store tolerates crash-faults and is still available and provides non-increasing latency in the face of this faults (up to $n-1$ faults for read availability and up to $n/2-1$ faults for write availability).
- **Reliability**: The operability of the event store is never or at least rarely interrupted.
- **Safety**: No node of the distributed event store will never return a false stream of events or false query results.
- **Maintainability**: In case of a failure, it is possible to recover from this failure in a short period of time and without compromising ongoing safety.
- **Integrity**: What has been written can not be changed.
- **Liveness**: Every write to the event store will eventually be persisted.
- **Partition-Tolerance**: Even in the event of network partitioning, the event store is available, and safety is never compromised.
- **Edge-Cloud Readiness**: The event store can be run on all three edge-cloud layers: embedded in another application, standalone on a single node or in a distributed setting, replicated and partitioned on multiple nodes.

Unfortunately, an ideal trade-off free solution can't be achieved. So, what trade-offs are acceptable for us? The trade-offs are described by the consistency model we decide for, which we will look at in the next subsection.

### Deciding for a Consistency Model {#sec:consistency-choice}

In this subsection, we elaborate our consistency model decision in detail, based on the previously listed requirements and use cases of a distributed event store, but also on industry best practices as we presented in section [@sec:previous-work]. Subsequently, we will mark the trade-offs to be made. We investigate the concept of consistency from the perspective of real-time and non-real-time event stream consumers and reveal new fundamental properties of consistency in this context.

<!-- Elaborate in this section in detail why we want strong consistency, also with examples. Later repeat the upper dependability requirements and mark the tradeoffs. -->

Throughout this discussion, we will use the following notation (some of the definitions are based on Koerber [@koerber2021accelerating] and Seidemann et al. [@seidemann2019chronicledb]):

\begin{definition}[Event Time]
The event time describes the time that an event occured, in contrast to the insertion time, that denotes the time an event was inserted into an event stream. We will denote timestamps in event time with $t$.
\end{definition}

\begin{definition}[Read Time]
The read time describes the real time of a client that reads an event stream. We denote timestamps in read time with $r$.
\end{definition}

\begin{definition}[Event]

An event $e_t$ is a tuple of several non-temporal, primitive attributes $(a_1, \dots, a_n)$ that resembles a notification about observations of facts and state changes from the real world at a timestamp $t$\footnote{There can be two events at the same timestamp $t$. In general, the subscript of $e$ denotes the index of the event in an event stream, based on event time ordering. We ignore this case in this work to simplify the following discussions and because it is not needed for understanding. Unless otherwise specified, $e_t$ denotes an event that happened at timestamp $t$ in event time.}. An event is immutable and persisted in an event stream. Not to be confused with the definition of an event in probability theory.
\end{definition}

\begin{definition}[Event Stream]
An event stream $E$ is a potentially unbounded sequence of events $\langle e_1, e_2, \dots \rangle$ that is totally ordered by event time. We call the domain of event streams $\mathcal{E}$.
\end{definition}

\begin{definition}[Time Window]
$E_{[t_i,t_j)}$ refers to a time window of an event stream, which is a slice of the stream containing all the events with timestamps in the time interval $[t_i,t_j)$ (excluding events at timestamp $t_j$).

A time windowing function $W_d$ applied to an event stream $E$ results in a non-overlapping sequence of such time windows. The time windowing function for the duration $d$ is defined as 
\begin{equation*}
    \begin{split}
        W_d(E) = \langle \\
        & E_{[t_1,t_1 + d)}, \\
        & E_{[t_1 + d,t_1 + 2d)},\\ \dots \rangle
    \end{split}
\end{equation*}

Note that we use a simplified definition of windowing here as we only care about tumbling time windows, which means that windows don't overlap.
\end{definition}

Furthermore, we use the following notation:

- $t$ reflects a single timestamp,
- $e_t \in E$ if the event $e_t$ is included in the event stream $E$,
- $E = \langle \dots, e_1 \rangle \xrightarrow{\mathtt{insert}(e_2)} \langle \dots, e_1, e_2 \rangle = E'$ describes a state transition of an event stream.

\todo{this whole discussion may also fit into fundamentals, and here only a reference?}

#### Consistency Perspectives

An event store introduces two different perspectives on consistency:

- The data perspective,
- The client perspective.

While the replication protocol could ensure strong consistency from the data perspective, the system could still look inconsistent from the client pespective due to the allowance of out-of-order events.

This is because strong consistency from the data perspective only cares about ordering of operations, therefore ordering of `insert` commands, which means it provides a strongly consistent ordering for all replicas by insertion time.

However, the client only cares about event time. Figure \ref{fig:event-store-consistency-levels} shows the different levels of consistency that can be achieved with strong consistency and allowance for out-of-order events. Figure \ref{fig:event-store-consistency-perspectives} illustrates the two perspectives in the event of out-of-order events arriving.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/event-store-consistency-levels.pdf}
  \caption[Consistency levels for event streams]{Consistency levels for event streams. a) With eventual consistency, no insertion order is guaranteed at all across replicas. b) With strong consistency on the data level, each replica shows the same consistent insertion ordering, but because of out-of-order events, it can still be inconsistent from the client perspective. c) Without out-of-order, strong consistency on the data level provides strong consistency also on the client level, as it guarantees that ordering by insert time $\equiv$ ordering by event time.}
  \label{fig:event-store-consistency-levels}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/event-store-consistency-perspectives.pdf}
  \caption[Insertion vs. event time]{Insertion vs. event time. While this insertion ordering in the event of out-of-order events still looks consistent from the data perspective, it looks inconsistent for the client, that only cares about the event time.}
  \label{fig:event-store-consistency-perspectives}
\end{figure}

If consistency from the client perspective is required depends on the applications that use the event store. The event store can be used as a _platform_ for other applications to build on. In this case, the consistency must be argued from the client perspective. It is not the state of the ChronicleDB state machine that is the critical factor, but the state of the applications that consume one or more event streams from the event store. One event may also be associated with state changes in multiple consuming applications, all with different requirements.

Thus, for event stores, we can describe consistency as the desired trade-off between insensitivity to the order of arrival of events and system performance [@barga2006consistent].

<!--
\myquote{``It is not the state of the ChronicleDB state machine that is the critical factor, but the state of the applications that consume one or more event streams from the event store.''}
-->

\todo{Later refer back to here when we introduce partial time bound consistency (i.e. real-time eventual consistency)}

##### The Problem with Out-Of-Order Events.

Out-of-order events naturally occur in real-world streaming systems. This can happen due to network delays, merging of streams with different timestamps and ordering semantics (such as one stream ordered by an "execution end" time while the other is ordered by an "execution start" time—events very often contain multiple timestamps, as is the case in ChronicleDB), or unordered intermediate result streams in event stream processing. ChronicleDB tolerates out-of-order events and allows clients to send events asynchronously, since they can be merged later into their correct position of the stream. In the embedded deployment of ChronicleDB, constant out-of-order rates are considered unlikely, but when it is deployed in a distributed setup, the probability is much higher.

Out-of-order events, if not handled properly, can violate consistency in real-time processing, as late-arriving events can cause non-monotonic derived aggregates to break as the causal chain is interrupted. We will elaborate on this in the course of the following sections.

In the shopping cart example from [@sec:stream-causal-consistency], if the `checkout` event is executed as soon as it arrives, the checkout result will include one item that should have been removed, as it missed the out-of-order `removeItem` event.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/windowed-consumers-ooo.pdf}
  \caption[Real-time stream consumers and out-of-order events]{Real-time stream consumers with time windows. a) As the single event consumer is already past the time of the out-of-order event, it will not read it anymore. b) The out-of-order event occurs when the consumer wants to read the time window, and is therefore still included in the result of the continuous query.}
  \label{fig:windowed-consumers-ooo}
\end{figure}

##### Consistency From the Client’s Point of View.

\epigraph{There’s no free lunch in the quest for analytics on both historic data and in real time.}{--- \textup{Jimmy Lin}}

What are the problems that are tried to be solved with event stores? There are both analytical and operational use cases. In the analytical use cases, real-time and historical data is combined for insight generation. In the operational case, events trigger new events, transactions and operations, such as in event sourcing.

In general, we distinguish between _real-time_ and _non-real-time applications_ when it comes to stream and event processing. This also introduces two seperate client consistency perspectives: the real-time client perspective and the non-real-time client perspective. A stream that appears consistent to a non-real-time consumer may not appear consistent to a real-time consumer. Commonly applied system architecture approaches provide both real-time and non-real-time processing to users and clients. Two popular architectures that provide this approach are the _Kappa architecture_ and the _Lambda architecture_. Both architectures allow for real-time and historical analytics in a single setup. The Lambda and Kappa architectures require that event processing reflects the latest state in both batch and real-time streaming layers (called the _speed layer_ in Lambda), hence requiring consistency. A merging layer allows to query the system for both historical and real-time data at the same time. At the heart of these architectures is an append-only data source—such as ChronicleDB, since it allows for both continous and ad-hoc queries [@marz2011lambda][^lambda-note] [@lin2017lambda].

[^lambda-note]: While we reference the blog post of Nathan Marz here which is described as the first mention of the Lambda architecture in 2011 [@marz2011lambda], what is in that post should be taken with a grain of salt, as some of the statements made here about "beating the CAP theorem" are simply wrong. Nevertheless, it is still worth a read, as it builds on similar ideas as "Immutability changes everything" by Pet Helland [@helland2015immutability].

What makes ChronicleDB special here is that it can act both as a stream provider and data lake (or warehouse). It unifies both OLTP and OLAP properties, making it possible to replace multiple lined up systems where data must be stored redundantly (such as Apache Kafka with a Hadoop or Cassandra instance for batch processing) in a Lambda architecture with a single ChronicleDB cluster. Note that currently, ChronicleDB is not able to replace Hadoop when it comes to big data batch processing (MapReduce) that does not fit into memory. ChronicleDB therefore makes an even greater companion in a Kappa architecture, where batch processing is simply done by streaming through historic event data.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/realtime-vs-nonrealtime-consumers.pdf}
  \caption[Real-time vs non-real-time event stream consumers]{Real-time vs non-real-time event stream consumers. a) The event stream "moves", while the consuming application stays fixed. b) The consumer moves along the event stream, probably in both directions, while the event stream stays fixed.}
  \label{fig:realtime-vs-nonrealtime-consumers}
\end{figure}

##### Real-Time Applications.

<!--
To illustrate, consider a financial services organization that actively monitors financial markets, individual trader
activity and customer accounts. An application running on a trader’s desktop may track a moving average of the value of
an investment portfolio. This moving average needs to be updated continuously as stock updates arrive and trades are
confirmed, but does not require perfect accuracy. A second application running on the trading floor extracts events from
live news feeds and correlates these events with market indicators to infer market sentiment, impacting automated stock trading programs. This query looks for patterns of events, correlated across time and data values, where each
event has a short “shelf life”. In order to be actionable, the query must identify a trading opportunity as soon as possible with the information available at that time; late events may result in a retraction. While a third application running in the compliance office monitors trader activity and customer accounts, to watch for churn and ensure conformity with SEC rules and institution guidelines. These queries may run until the end of a trading session, perhaps longer, and must process all events in proper order to make an accurate assessment. These applications carry out similar computations but differ significantly in their workload and requirements for consistency guarantees and response time [@barga2006consistent]
-->

In (near) real-time systems, the data is read and processed incrementally, and immediately after it is inserted into the event store. Oftentimes, the result of the real-time analysis of the raw events are aggregates that are served again to other applications that build upon this aggregates, or to end users to query this data, thus relying on the consistency of this data through multiple levels.

Such systems can be seen in a way that the stream "moves", while the consuming application stays fixed. With the stream moving through the consumer, it sees all recently appended events in near-real-time, at least with some delay. This is shown in figure \ref{fig:realtime-vs-nonrealtime-consumers} a. Usually, the stream is not scanned event by event, but in time windows, as illustrated in figure \ref{fig:realtime-vs-nonrealtime-consumers}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/windowed-consumers.pdf}
  \caption[Real-time stream consumers with time windows]{Real-time stream consumers with time windows. a) Consuming one event at a time. b) Consuming multiple elements at the same time with time windows.}
  \label{fig:windowed-consumers}
\end{figure}

###### Pub/Sub Systems.

_Publish/subscribe_ (pub/sub) systems are messaging systems that allow for clients to publish events (or messages) into _topics_, while other clients can subscribe to the topics of their choice (often with additional filtering and routing capabilities). Queries to such systems are usually stateless and do not provide powerful computational queries other than filtering. Pub/sub systems like Kafka or RabbitMQ often allow users to select the consistency level on a per-queue basis. 

They often distribute a single message to a magnitude of clients, which is called broadcasting. One example for such messages are mobile push notifications. They rarely need to be sent in order, and arrival guarantees are only eventual in a certain timeframe, but this allows to replicate one message to millions of devices with very low latency. This is an example of optimistic replication with eventual consistency, as the publisher of such a message does not require recipients to acknowledge the receipt—which is nearly impossible as it is highly probably that many of the target mobile devices on the subscribers list are not available. Another example for eventually arriving messages are SMS messages, but in this case, their receipt is acknowledged, and the SMS is sent to a single recipient rather than being broadcasted.

But those systems can also send messages with high safety and consistency guarantees, where both the shipping and receiving of the message is confirmed by a quorum, and the message order is guaranteed at least if a single consumer reads the message. If messages are distributed across multiple consumers (e.g., to distribute load to workers), ordering can not be guarenteed—that is, processing order. In the case of a processing failure, to achieve strong ordering guarantees, the whole queue must be blocked and possibly already started processing stopped and reverted. That is some behavior that is rarely desired.

Due to the lack of more powerful computational queries (which is in fact not their target use case; these systems are designed to route events rather than process them), one often sees these systems combined with event sinks (data lakes), either on the producer or consumer side, or both, which in this case could be ChronicleDB.

###### CQRS and Event Sourcing.

In use cases such as _Command and Query Responsibility Segregation_ (CQRS) with event sourcing, commands that are logged in an event store will always be written (and read for consecutive commands) in a strongly consistent manner. However, on the query part, an aggregate is derived (called a _projection_) and shown, that gets stale very fast, due to the high velocity of events written to such an event store. What a user sees on their screen may be already outdated, depending on the domain of that data. This means that users can only be served with eventually consistent snapshots of the current state—even if the UI is reactive (i.e. events are pushed, e.g., via web sockets, to notify the UI about a potential state change), because without locking the stream in the period of loading the most recent state, the state could already have been updated. This happens, for example, in high-volume stock trading, where online brokers try to serve the current price and order book as recent as possible, but this heavily relies also on the network latency of the user. When placing a market order, it is filed strongly consistent against the most recent price at this time, which the user may haven't yet seen, while some other brokers freeze the last price seen by the user for some seconds to guarantee exactly this price - which is then to be placed on a possibly already updated market.

###### Event Stream Processing.

ChronicleDB is designed to run side-by-side with _event stream processing_ (ESP) systems. 

Traditional ESP applications handle a single stream of events that arrive in the correct event order. An example would be algorithmic trading, where a simple ESP application could analyze a stream of price data and decide whether to buy or sell a stock, such as in figure \ref{fig:ohlc-ooo-problem}. ESP applications typically do not involve event causality or event hierarchies.

Typical event stream processing queries look relatively similar to SQL. In the _Continuous Query Language_ (CQL), a query looks similar like this:

```
SELECT AVG (O.total)
FROM Orders O [RANGE 10 Minute, SLIDE 10 Minute] 
WHERE O.product = productId
```

This query returns a moving average of the order volume for a certain product over a tumbling time window of 10 minutes.

In ESP, we differentiate between in-order processing and out-of-order processing.

###### IOP (In-Order Processing) vs OOP (Out-Of-Order-Processing).

_In-order processing_ (IOP) systems require strong consistent ordering of the event streams that are continously read. In case the events from the event store are to be fed into such systems, the event store must ensure strong consistency.

This approach fits all use cases where strong consistency is required, such as use cases from the financial domain or other real-time decision making cases with high business criticality.

On the other hand, _out-of-order processing_ (OOP) systems do tolerate such events arriving with a delay at the processing layer. This can speed up the processing of such systems tremendeously, especially in distributed event processing, in a similar fashion that eventual consistency speeds up the replication in a distributed data storage system and reduces overall latency. The windowing techniques of such systems allow for a timeframe where out-of-order events can appear without violating the results of the event processing. They can be used for all real-time applications where the use case allows for a degree of inconsistency. Once the use case allows for OOP, this approach can "significantly outperform IOP in a number of aspects, including memory, throughput and latency" [@li2008out]. 

An out-of-order processing system continues to process a stream until a specified condition applies. While the ordering guarantees of IOP allow the progress of stream processing to be acknowledged after each processed in-order event, in OOP, progress is confirmed only when such a condition occurs. These conditions can be expressed using _punctuation_.

Punctuation works similar to punctuation in written language and marks the end of a substream (analogous to the full stop marking the end of a sentence). This allows us to view an infinite, unbounded event stream as a mixture of finite streams [@tucker2003exploiting]. Inside of such a sequence, order does not matter. No further event violate previously received punctuation. OOO is allowed to happen in such a "sentence", but not across sentences.

With punctuations, only the punctuations must be ordered, and events between the punctuations are allowed to be unordered, but must be the same set of events (no out-of-order events are allowed to come after their punctation). To identify the correct set of a sequence, the punctuation message `punctuate({e_1, e_2, \dots})` contains the ids of those events to expect in the sequence. This is similar to the "weak consistency" approach, where only expliticitly synchronized parts must appear in order, and everything in between can be unordered. This means that for all events $e$ with $e.t \geq p_1.t \wedge e.t < p_2.t$, $p_1 \prec e \prec p_2$, while any order between events is valid, where $p_1$ and $p_2$ are the punctuations around $e$.

In ChronicleDB, a similar approach could be adopted to mitigate the out-of-order problem. With this approach, events would not be required to be inserted in event order, while the punctuation allows for strong ordering guarantees in the aftermath. It would also require event producers to support punctuation. It could either be implemented right in the replication protocol by introducing it in the log and weaking the consistency guarentees between punctuation, or in the event streams themselves. We haven't found any literature on this topic yet, nor have we figured out how this could be implemented (what if punctuations arive themselves out-of-order?).

\todo{Draw chart for the shopping cart case with punctuation}

Not all use cases can be served with OOP, as implementing and applying punctuation is far from trivial and can not be used in every situation. Nethertheless, this approach comes in handy for all real-time applications where no strong consistency is needed, such as calculating moving averages and similar aggregates.

##### Non-Real-Time Applications.

In non-realtime applications, the consumer moves along the event stream, while the event stream stays fixed. The consumer can move in both directions, scanning the stream. This is shown in figure \ref{fig:realtime-vs-nonrealtime-consumers} a.

In such applications, out-of-order events are not a big threat, as in general, they arrived a long time ago in the event store when a query is run on historic data. Even more, the eventual consistency model is oftentimes sufficient since it is only necessary that the write operations converge at some point in time in the event store. 

Examples for non-real-time applications are _Business Intelligence_ (BI), data science, process mining or machine learning. We have drawn more examples in figure \ref{fig:chronicle-edge-architecture}. Also, using an event store as a data lake to store derived events after processing does not require strong consistency. 

Historic queries can be used to detect faulty results from real-time queries due to delayed events, as the results of historic queries have a higher chance to be consistent. To repair such a faulty result, it requires the developer to write compensation logic in case the real-time layer already caused executions based on incomplete or stale data, similar to the _Saga pattern_. But, this is only limited to actually compensable operations. For example, how should a business compensate financial transactions in the real world that are made based on wrong assumptions due to incorrect data?

\todo{Finish this section}
-->

<!--
TODO CEP ESP stuff here
-->

##### Complex Event Processing.

_Complex event processing_ is a method to derive new conclusions from event streams in real-time. It applies pattern matching to derive new events, called _complex events_, from a sequence of raw events 
[@buchmann2009complex]. From a domain-driven perspective, these complex events are _derived aggregates_.

Compared to ESP, complex event processing systems patterns are generally detected across multiple streams of different domains. We illustrate this in figure \ref{fig:complex-event-processing}. Complex event processing systems can detect patterns across event streams, including event occurrence and non-occurrence, and queries can set complex constraints on timing. An example for a pattern that matches against the non-occurrence of events is one that detects customers that did not respond to an email within a certain time frame.

To run patterns that rely on the non-occurrence of events (thus, deriving non-monotonic aggregates), a strong consistent ordering during the pattern matching windows is required. In general, CEP systems do not provide strong consistency guarantees, which puts the demand for ordering on the event producer (or, in this case, the intermediary store). We have illustrated what can happen when events appear delayed in figure \ref{fig:ohlc-ooo-problem}. 

Eventual consistency for CEP is only allowed for monotonic problems, because in this case new information does not change the already derived facts. Additional facts can only arise from discovering additional derived events: the output (computed events) grows monotonically with the input (factual events).

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/complex-event-processing.pdf}
  \caption[Complex event processing with event correlation]{In complex event processing, events from multiple streams are fuzzily correlated to derive monotonic aggregates}
  \label{fig:complex-event-processing}
\end{figure}

Complex event processing is also a key component of event-driven systems. Instead of dealing with raw events, microservices consume complex events (such like a signal) derived from these raw events, and they must do this in a reliably, safely manner.

<!--
TODO compare with what
- https://fauna.com/blog/why-strong-consistency-with-event-driven
- and https://drops.dagstuhl.de/opus/volltexte/2019/10307/pdf/LIPIcs-ICDT-2019-5.pdf
says
-->

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/ohlc-ooo-problem.pdf}
  \caption[Algorithmic trading as an example for strong consistency]{Algorithmic trading as an example for strong consistency requirements. a) A candlestick pattern was detected in the trading events in a order book, causing a buy alert to be sent (and executed). b) At a later time, the past candlesticks are displayed differently due to some out-of-order events, so that the buy alert is now wrong - but it is already too late. This reduces trust in this hypothetical trading platform and its reputation.}
  \label{fig:ohlc-ooo-problem}
\end{figure}

##### Other Use Cases.

The event store can be simply used as a replication engine itself (similar to ZooKeeper) by running it next to a distributed system to store and replicate the commands of this system. It then turns out to be a replicated log with some extensions (timestamps denoting event-time, aggregates and powerful indexing). It could also be used to create event-sourcing system or to create a pub-sub architecture. For the consuming application(s), the consistency of the system is then also dependent on the allowance for out-of-order events. If the use cases of the consumers allow with eventual consistency, then a eventual consistent implementation that also allows out-of-order events is sufficient. Else, the system must be linearizable, providing strong consistency, which also forbids any out-of-order events.

##### Conclusion.

So far, it appears that some applications require a strict notion of correctness that is robust in terms of the event order, while others place more emphasis on high throughput and low latency. The perspective of the application is more important than the perspective of the event store when it comes to a consistency decision. We need to remember that a recently inserted event may be associated with state changes in one or more consuming applications, as in figure \ref{fig:multi-states-stream}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/multi-states-stream.pdf}
  \caption[Multiple applications deriving state from a stream]{Multiple applications that derive their state from subsets of an event stream}
  \label{fig:multi-states-stream}
\end{figure}

#### Causal Consistency {#sec:stream-causal-consistency}

In an event store, causal consistency must be evaluated from a client perspective, since the only write operation is the `insert`, and there are no mutations of existing events.

One example for causal relations between events comes from the event sourcing domain: remember the shopping cart example from section [@sec:calm]. Even if `insertItem` and `removeItem` commands can be designed to be monotonic, thus non-causal from a data perspective since their order does not matter for each other, a `checkout` command is causally related to all these previous commands for this particular shopping cart session, thus it must be ordered after this commands. In event-sourcing systems, such commands should never arrive out-of-order if executed in realtime, because it could render the derived aggregate wrong that defines the items included in the checkout. We illustrated this in figure \ref{fig:checkout-example}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/checkout-example.pdf}
  \caption[Causally dependent operations in event sourcing]{Causally dependent operations in event sourcing, illustrated with a shopping cart example. a) While the order between some domain events does not matter, the order between them and another command is important, implying a causal dependency. b) If causally related events arive eventually, such as with out-of-order events, an already executed command may have produced wrong results.}
  \label{fig:checkout-example}
\end{figure}

The causal relation of events can depend on attributes of the events and their domain (note that we are not discussing causal consistency from the perspective of write-read command chains as in the actual definition from section [@sec:consistency], but from real-world causality). For example, if a single stream contains events from multiple sensors, and these sensors do not influence each other, a sensor ID can be used to segment the stream. The happens-before relation applies to all events of such a segment, thus the events in the segment are causally related. This is illustrated in figure \ref{fig:causal-stream-split}. In another example, these sensors may be measuring temperatures in different regions, but the client is interested in finding correlation in temperature changes between these regions, but also with temporal causality: does the change in temperature at one location affects the other?

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/causal-stream-split.pdf}
  \caption[Causal relation in event streams]{Two causally consistent segments of event streams. a) Both segments reside in a single event stream. b) The segments are split up into their individual streams, that are now strongly consistent.}
  \label{fig:causal-stream-split}
\end{figure}

But there can also be a causal relationship across streams with different schemas, depending on the domain, as illustrated in figure \ref{fig:causal-across-streams}. For example, consider an event stream that contains temperature measurements while another contains humidity measurements. The events between the streams are assumed to be causally related: A rise in temperature causes a (perhaps delayed) rise in humidity in the same region. Finding such patterns is a typical use case of complex event processing (CEP). Note that causal consistency is not sufficient to detect patterns that include the non-occurrence of events: causal consistency only covers the observation of existing events. 

Since this is up to the user, we recommend to design streams in such a way that the causal relationship is kept coharently on a per-stream base. That is, if a stream can be segmented in a way that the segments are no longer causally related, those segments should be streams of their own, and if events appear to be causally related across streams, it may be appropriate to merge the streams, provided their schemas are compatible. This also increases the performance of the system, as `insert` operations of unrelated events do not need to be coordinated. However, some other properties must be taken into account, such as data locality (keep seperate streams close to where they are created), edge-cloud design (merge streams of two sensors afterwards on the edge or cloud if they are causally related) and sharding (find a balance between causality and scalability; i.e. if the throughput limit of one stream has already been reached, it makes sense to partition it even if events in these shards are causally related).

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/causal-across-streams.pdf}
  \caption[Causal relation across event streams]{Two event streams, i.e. for different domains. a) Inter-stream causality between events of the two streams. b) Due to out-of-order events, intermediate states of the streams violate causal consistency.}
  \label{fig:causal-across-streams}
\end{figure}

##### Out-Of-Order Events and Causality.

Out-of-order events cause violations of intermediate states of streams, as the causality chain can be broken. This is illustrated in figure \ref{fig:causal-across-streams} b. Depending on the consuming applications, this can be a serious problem. Figure \ref{fig:ooo-consistency} shows a generalized picture of how non-monotonic aggregates, which are apparently causally dependent on the source events, can be violated by out-of-order events. Such faults introduced by delayed events would require read-repair code, such as applying the SAGA pattern, to be mitigated in the aftermath, which is prone to more errors to be introduced, and even not always possible.

The following sections describe the out-of-order problem in more detail and explain how it can be mitigated.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/ooo-consistency.pdf}
  \caption[Out-of-order events breaking non-monotonic aggregates]{For non-monotonic derived events or aggregates, out-of-order events must be prohibited, as they could render the derived events wrong: what was already observed must not change. a) A non-monotonic derived aggregate is created based on previous facts (events). b) When observing the event stream later again, an out-of-order event occured which invalidated the previous derived aggregate and created a new aggregate.}
  \label{fig:ooo-consistency}
\end{figure}

##### Conclusion.

Many use cases require a distributed event store to be at least causally consistent, but also with respect to the user perspective. Unfortunately, the causal relation between real-world items is unknown to a replication protocol; a protocol only knows about writes and reads and the causal dependency between these commands. This must be mitigated by the user by proper design of the event stream partitions.

#### Coordination-Free Replication

One of the first questions to ask when thinking about a consistency model is, if we can achieve coordination-free replication—which means, eventual consistency is sufficient. We ask ourselves if event stores are monotonically growing, which is the requirement for coordination-free replication as described in the CALM theorem (see subsection [@sec:calm]).

So far we have seen that for many use cases, causal consistency is needed, which means that coordination-free replication is not possible in this cases. In this subsection, we elaborate this in more detail.

Currently, there is only one write operation in the event store: the `insert` operation. We consider ChronicleDB to be append-only, therefore ignoring the case for deletes. In fact, most cases of deletes are better represented as creating new events that invalidate or reverse the state change of the the initial event (if deletes are actually required, such as to adhere to the GDPR, it gets more complicated).

From the point of view of a single event, the `insert` operations is associative, commutative and idempotent, all the properties that allow for coordination-free replication with eventual consistency:

- Inserting a single event at a given timestamp will insert it at this timestamp no matter if the insert was executed before or after some other insert.
- Using mechanisms like a session identifier, repeated inserts of the same event at the same timestamp could be ignored, rendering the insert operation as idempotent.

But from the point of view of the overall state, which is a ordered streams of events, the insert is not associative and commutative: Running inserts in different orders will result in different intermediate states of the stream, even if the final state is the same. Imagine inserting two events $e_1$ and $e_2$ in any order, this could result in two intermediate states $\langle \dots, e_1 \rangle$ and $\langle \dots, e_2 \rangle$, while the final state is $\langle \dots, e_1, e_2 \rangle$. $\langle \dots, \mathtt{insert}(e_1), \mathtt{insert}(e_2) \rangle$ leads to a sequence of states $\langle \dots, e_1 \rangle \to \langle \dots, e_1, e_2 \rangle$, while $\langle \dots, \mathtt{insert}(e_2), \mathtt{insert}(e_1) \rangle$ leads to an inconsistent sequence of states $\langle \dots, e_2 \rangle \to \langle \dots, e_1, e_2 \rangle$. Without any other hints, applications may not be able to distinguish between an intermediate and final state, nor should they be prompted to do so. We could lower the requirements and help applications distinguishing between an intermediate (unordered and inconsistent) and final state with punctuation as shown before in this chapter. We then let query operators order punctuated sequences before returning them, as shown in figure \ref{fig:query-consistency}). Derived aggregates that are only allowed to be created for punctuated sequences are safe then. But introducing punctuations is far from trivial. For instance, a punctuated sequence would look like the following: $\langle \dots, \mathtt{insert}(e_2), \mathtt{insert}(e_1), \mathtt{punctuate}(e_1, e_2) \rangle$ leads to a sequence of states $\langle \dots, e_2 \rangle \to \langle \dots, e_1, e_2 \rangle \to \langle \dots, e_1, e_2, \cdot \rangle$, where only the last state is considered to be a final state.

##### Conclusion.

Depending on the use case, an event store is not monotonically growing. The derived aggregate resulting from the event streams may or may not be a monotonic aggregate, depending on the application using the aggregate. With non-monotonic aggregates, the order of the events, and also the order of the appearance of the events, matter.

The consistency requirements to the event store are heavily dependent on the actual use cases of the consuming applications.

At least causal consistency is required by a subset of real-time applications that rely on non-monotonic aggregates. Punctuations could be a way to introduce causal consistency and to reduce coordination inside of punctuated sequences. Non-real-time applications will probably not experience missing events due to out-of-order or eventual consistency, so eventual consistency is oftentimes sufficient.

#### Time-Bound Partial Consistency {#sec:time-bound-partial-consistency}

When out-of-order events can not be avoided, or the maximum throughput should be increased (and latency reduced) by introducing lower consistency constraints, it is worth to think about _time-bound partial consistency_. This means that for different sequences of a event stream, different consistency promises could be made. This allows to bypass the latency-consistency trade-off of the PACELC theorem, at least practically. Figure \ref{fig:time-windows-consistency} shows that we for older time windows, consistency increases—in general in an exponential nature—and we want to take advantage of this property.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/time-windows-consistency.pdf}
  \caption[Time windows and consistency]{Illustrative example for increasing consistency for past time windows}
  \label{fig:time-windows-consistency}
\end{figure}

##### Convergence and Eventual Consistency.

As we have shown, strong consistency can be relevant for real-time applications, contrary to the low-latency requirements for real-time processing. Non-real-time applications will probably not experience missing events due to out-of-order or eventual consistency, as the probability that time windows of the event stream provide a consistent view—i.e., that the have converged—increases when we move along the stream in descending time order, and it also increases with a decreasing window size.

By investigating the convergence behavior of a system, we can make better assumptions on its practical consistency. Therefore, we will describe consistency and convergence in the following definitions:

\begin{definition}[Ground Truth]\label{def:ground-truth}
$E^G_{[t_i, t_j)}$ describes the total series of events that happened during the time interval $[t_i, t_j)$ and that are to be persisted in the event store. We call it the ground truth of events during this interval.
\end{definition}

We expand our definition of time windows:

\begin{definition}[Time Window at Read Time $r$]
$E^r_{[t_i,t_j)}$ refers to the time window $E_{[t_i,t_j)}$, observed at read time $r$. The read time is the real-time where the time window is observed, i.e., by a consumer.
\end{definition}

\begin{definition}[Consistency]
With $C(E^r_{[t_i,t_j)})$, $C : \mathcal{E} \to [0, 1] \in \mathbb{R} $ we denote the consistency of the given time window at real-time $r$. It denotes the ratio of existing and known events that are in the time window of the stream at this given point in time:

$$ C(E^r_{[t_i,t_j)}) = \frac{|E^r_{[t_i, t_j)}|}{|E^G_{[t_i, t_j)}|} $$
\end{definition}

Eventual consistency guarantees liveness: data will eventually converge in the absence of updates. We say that a stream converges if

\begin{equation}\label{eq:ground-truth-convergence}
\lim_{r \to \infty} E^r = E^G
\end{equation}
where $r$ refers to read time. The same applies then to each possible time window:
\begin{equation}\label{eq:ground-truth-window-convergence}
\lim_{r \to \infty} E^r_{[t_i,t_j)} = E^G_{[t_i, t_j)} \hspace{16pt} \forall t_i, t_j, t_i \leq t_j
\end{equation}

Based on \ref{eq:ground-truth-convergence}, it is

$$ \lim_{r \to \infty} C(E^r_{[t_i,t_j)}) = 1 \hspace{16pt} \forall t_i, t_j, t_i \leq t_j $$ 

In the absence of updates.

We claim that this is also the case for a fixed time window during consistent updates up to a certain maximum throughput.

We need to describe convergence first:

\begin{definition}[Convergence Rate]
The convergence rate $\mathtt{Conv}$ of an event stream time window $E_{[t_i,t_j)}$ in the real-time interval $[r_n, r_m]$ describes the rate of events of the ground truth $E^G$ inserted into the stream during the interval, compared to new events that arrived in the ground truth.

The convergence rate is defined as

$$ \mathtt{Conv}(E_{[t_i,t_j)},[r_n, r_m]) = C(E^{r_m}_{[t_i,t_j)}) - C(E^{r_n}_{[t_i,t_j)}) $$

Hence, it can be generalized to the convergence rate at read time $r$

$$ \mathtt{Conv}(E_{[t_i,t_j)}, r) = C'(E^r_{[t_i,t_j)}) $$

A local convergence rate $> 0$ means the stream is about to converge, while a local rate of 0 describes a stream that does not converge at the moment (either it already has converged or the gap size between inserted and occurred events stays consistent), while a local rate $< 0$ describes local entropy (there are more events occuring than the system is able to handle).
\end{definition}

We observe that a fixed time window becomes eventually consistent if and only if the average convergence rate $\geq 0$.

$$ \lim_{r \to \infty} E_{[t_i,t_j)}^r = E_{[t_i,t_j)}^G \longleftrightarrow \mathtt{Conv}(E_{[t_i,t_j)},[r_0, r] )\geq 0 $$

##### Convergence and Scaling.

In real-time systems, writes and reads occur continously. Until writes stop, an event stream will never converge, as there are always new events occuring that are not yet written to the store.

Looking at a fixed time window, it will only never convergence if and only if delayed (out-of-order) events are never written to them. If this happens, there was either a fault that prevented the event from appearing, a extremely long network delay, or, the most problematic case, the overall throughput is too high, lowering the convergence tremendously. Regular convergence behavior of a fixed time window is illustrated in figure \ref{fig:convergence}. We will see in the next paragraphs, that we can describe a probabilistic guarantee for time windows to converge. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/convergence.pdf}
  \caption[Convergence of a fixed time window]{Looking at the time window at the head of the stream, we see that it converges after a period of inconsistency when events, including out-of-order events, arrive at the window}
  \label{fig:convergence}
\end{figure}

Continously looking at the head of the stream (the most current time window), under continous writes it will never convergence. If the throughput is acceptable, we will observe a steady convergence rate of 0 (if we write in batches, we may see some stairs-shaped curve). We use this observation in the following paragraphs to describe a way to improve write latency and give better consistency guarantees for reads.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/convergence-head.pdf}
  \caption[Convergence of the head of the stream]{Simplified illustration of the convergence behavior of the head of an event stream. Continously looking at a time window at the head of the stream (i.e. moving it while we move in real-time), we see that the head of the stream never converges until writes to the stream stop. We can identify throughput spikes. If the average convergence rate is 0, the throughput at this stream and node is healthy.}
  \label{fig:convergence-head}
\end{figure}

A continously negative convergence rate is a cause for concern and signaling that the current throughput may be too high for the system to handle, causing entropy between nodes and clients and finally slowing down the system. This could serve as an ideal signal for elastic auto-scaling: When entropy is detected, new nodes can be provisioned and added to the cluster to handle and balance the additional load. But since only the producing client knows the ground truth of events, it must be responsible to detect convergence by comparing what is written and incoming write acknowledgements. This won't work in an asynchronous fire-and-forget mode, though.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/consistency-function.pdf}
  \caption[Illustration of consistency functions over time windows]{Two illustrative consistency functions over time windows for a high and a low throughput scenario. Because out-of-order events are distributed in an exponential manner across a stream as the chance for a time window to converge increases over time, the consistency increases when we travel back in the stream. The rate of out-of-order events increases with throughput—and also with the level of concurrency.}
  \label{fig:consistency-function}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/eventually-consistent-stream.pdf}
  \caption[Event stream with out-of-order-events]{Event stream with out-of-order events arriving later in real time, thus only providing eventual consistency to the client. In general, out-of-order events are distributed in an exponential manner, i.e., out-of-order events happen closely to their source. a) The stream is missing some events from the ground truth that are not yet sent to and committed by the store. With decreasing event time, the stream becomes more consistent. b) Eventually at a later point in real time, a past portion of the stream converged, while for a recent time period out-of-order events may still arrive.}
  \label{fig:eventually-consistent-stream}
\end{figure}

##### The Inconsistency Window.

Time-bound partial consistency can be achieved by ensuring that out-of-order events (or all delayed events in the case of eventual consistency) are written only in a certain time window, called the _inconsistency window_. We sketched to inconsistency window in figure \ref{fig:convergence}. Depending on the use cases and business criticality, this can be of high interest, as the cost of replication (such as latency) can be massively reduced.

Let the consuming client decide on the level of allowed inconsistency by specifiying the latest timestamp $t_{\mathtt{now}-n}$ to that events are returned. This timestamp is referred to as the _heuristic watermark_ [@lin2017lambda]. All events after this timestamp (thus, more recent entries) are not returned to the consumer, as this time window is allowed to be inconsistent.

With out-of-order events, and especially with eventual consistency, there is no guarantee for consistency. This means that the timestamp is chosen in a way that $P(C(E_{[t_{\mathtt{now}-n}, t_{\mathtt{now}}]})) \leq 1$, but $P(C(E_{[t_i, t_j]})) \approx 1$ $\forall t_i, t_j, t_i < t_j < t_{\mathtt{now}-n}$. We denote $E_{[t_{\mathtt{now}-n}, t_{\mathtt{now}}]} = E_{|n|}$ as the _inconsistency window_.

The inconsistency window is either not allowed to be read by consumers or the consumers must explicitly require it, knowing that they may receive inconsistent results.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/inconsistency-window.pdf}
  \caption[Inconsistency window]{Illustration of the inconsistency window. The events in this window are not visible to consuming clients because out-of-order events are likely to appear here.}
  \label{fig:inconsistency-window}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/ooo-probability.pdf}
  \caption[Out-of-order probability]{Illustration of the out-of-order probability for the inconsistency window and the remainder of the stream.}
  \label{fig:ooo-probability}
\end{figure}

\begin{definition}[Out-Of-Order Probability]
The out-of-order probability $P_{\mathtt{OOO}}(E_{[t_i, t_j]})$ describes the chance for one out-of-order event to appear in the stream in the time interval $[t_i, t_j]$.

We illustrate this in \ref{fig:ooo-probability}.
\end{definition}

\begin{definition}[Inconsistency Probability]
The inconsistency probability of a time window describes the probability of that window to be inconsistent, thus $P_{\mathtt{IC}}(E_{[t_i, t_j]}) = 1 - P(C(E_{[t_i, t_j]}))$.

In an event store with strong consistency, this is equal to the out-of-order probability.
\end{definition}

The out-of-order and inconsistency probabilities can be estimated by continously observing an event stream and are served on a per-stream base. In the non-faulty case, the maximum size of the inconsistency window can also be estimated based on communication delays, system load, and the number of servers involved in replication [@lindstrom2014real]. ChronicleDB supports calculation of the _maximum delay_ for events on the fly. It describes the maximum amount of events that are present between an out-of-order event and the place that it was supposed to be inserted at according to application time [@seidemann2019chronicledb]. This allows to derive the inconsistency window with built-in methods.

With this, rather then deciding for the heuristic watermark, a real-time application could agree on a tolerated inconsistency probability. This means that they would define the minimum required consistency probability $p_{C}$ for the remainder of the stream (the sequence of the stream that is allowed to be read) by configuration. This is equal to the the minimum inconsistency probability that the inconsistency window must cover, making it simple to argue about the length of the inconsistency window (and to derive the heuristic watermark):

\todo{If time, denote this as an algo}

We continously observe and register the delays of out-of-order events in the stream and set the window size in a way that it covers the $p_{C}$ percentile of these delays, including an additional tolerance area (such as 20% of this timeframe). We choose the percentile so we can exclude outliers, as shown in figure \ref{fig:ooo-distribution}. We don't recommend to choose a value of 100%, as if any outlier occurs (due to a node fault or other exceptional delays), it would render the inconsistency window too long, thus denying any near-real-time properties. 

<!--
This is still not suitable for every use cause, though. For instance, a consistency probability of 99.999% would mean that there is a chance of one in a million that there is a out-of-order event arriving before the inconsistency window, and due to the exponential distribution of the delay of event insertions, this is nearly the chance that it is read in recent time windows. In high-volume financial transactions, this is still not tolerable.
Actually, by using the percentile of delays, we would cover even more out-of-order entries. 
-->

This estimate works well enough when the out-of-order events are exponentially rather than uniformly distributed, i.e., out-of-order events happen rather closely to their
source. This is generally the case. We illustrated this in the figures \ref{fig:ooo-distribution} and \ref{fig:eventually-consistent-stream}. We advice to continously monitor the arrival of out-of-order events (or, in eventually consistent streams, the time for events to arrive in general) to be able to react on expectional patterns that might render the time windows before the window inconsistent. The inconsistency window should be calculated on a per-query base, as shown in figure \ref{fig:query-consistency}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/query-consistency.pdf}
  \caption[Potential query consistency module]{Potential query consistency module for ChronicleDB. An intermediate consistency module could provide practical ordering guarantees for queries by adding additional delay on a per-query base that incorporates the maximum delay for out-of-order entries in the inconsistency window. Practically, the module holds the input stream in a buffer until an output can be generated that adheres to the desired consistency level. Applications must therefore make trade-offs between delay and consistency for each query.}
  \label{fig:query-consistency}
\end{figure}

When deciding for the consistency probability, clients need to find a balance that allows for a low chance of inconsistent entries to be read, while available data should be as recent as possible (near real-time).

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/ooo-distribution.pdf}
  \caption[Distribution of out-of-order events]{Distribution of out-of-order events in an event stream, relative to the current timestamp}
  \label{fig:ooo-distribution}
\end{figure}

##### Guaranteeing the Inconsistency Window.

In theory, the whole stream is still eventually consistent from the clients view, but in practice, inconsistent reads outside the inconsistency window should be very rare. The consistency probability could be turned into a guarantee by covering it in SLAs, for example. Or, we could even harden this property by requiring the following guarantee: All events must be eventually written during the inconsistency window. This would mean that we would reject any out-of-order entry that reaches the event store with a delay larger then the inconsistency window, sending a `nack` to the event producer and making the producer responsible to compensate for the missing event. 

##### Conclusion.

By investigating the convergence behavior of a stream, we can argue about different levels of consistency throughout one stream from the clients perspective. By continously observing convergence and adjusting the inconsistency window on a per-stream basis, we can improve the latency and consistency while providing near-realtime reads to consumers. From a theoretical point of view, this still does not provide a strong consistency guarantee, but the probabilistic guarantees may be sufficient for several real-time use cases where write latency is critical. 

#### Buffered Inserts {#sec:buffer-theory}

One solution to cope with out-of-order events is to buffer `insert` operations temporarily before writing them to the stream. The buffer is flushed and all events written in a batch once its content reaches a certain size or after a certain timeout after the last write. On flush, events in the buffer are ordered by event time, instead of insertion time, ensuring consistency for the batch write. This also decreases latency introduced by the replication protocol dramatically, since the network roundtrips are reduced, which we will elaborate later in this chapter.

\todo{Refer to this section later again in implementation}

The the buffer size and timeout can be chosen in a way that it covers the max delay of out-of-order events—respectively the inconsistency window. This has a similar effect to implementing the inconsistency window in the query processor, with the difference that events in the buffer are only hold in-memory on a single machine instead of being written directly to the event stream, thus causing the risk of data loss of the events in this buffer.

The effect of a write buffer on the overall consistency is shown in figure \ref{fig:consistency-function-buffered}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/consistency-function-buffered.pdf}
  \caption[Consistency function using a buffer]{Using a buffer to order events in advance and write them in batches increases the consistency dramatically in the face of out-of-order events and concurrency.}
  \label{fig:consistency-function-buffered}
\end{figure}

While a buffer increases the real-time consistency from the client perspective (the chance is higher that events arrive at consumers ordered by event time), it reduces the data consistency from linearizable to sequential, since after a write, an event can not be immediately read. But, when it is read in real-time, it is ordered (with a high chance). Acknowledging each single write to the stream to the producer is challenging: this would require callbacks that must be managed somewhere, e.g. in the replication log. We currently only acknowledge a successful write to the buffer, but this does not guarantee a successful write to the quorum. Only the acknowledge of a flush of the buffer and the execution of the batch of events lets the client know of a successful write to the store, but this adds a lot of latency to this acknowledgements. In case of unavailability or faults, the client will not get an immediate insight into this.

#### Consistency Across Partitions

Keeping writes consistent across partitions, and especially across streams of different schemas, is challenging, especially when each partition is covered by its own instance of the replication protocol. This requires another layer of coordination across partitions, which increases the overall latency. It is also only relevant for streams with inter-stream causality (see subsection [@sec:stream-causal-consistency]).

Instead of linearizing writes across partitions (which anyway only ensures ordering in insertion time), it is better to ensure inter-stream ordering on query time. If there is causality across streams, this becomes only relevant on queries on multiple of such streams, and should be handled by the distributed query processor.

Note that even popular event stores and similar systems often do not support strong consistency guarantees across partitions, such as in Azure Event Hubs [@microsoft2022eventhubsconsistency].

#### Conclusion

What have we decided for? The previous discussion has shown that the consistency model applicable heavily depends on the use case of the event store: event sourcing requires at least causal consistency, ESP relies commonly on strongly consistent data and in CEP, pattern matching also works on monotonicly derived aggregates on eventual consistent data. Non-realtime applications are usually satisfied with eventual consistency. This shows that the event store is not the application, but a platform, inheriting its consistency requirements from the consumers. We must keep in mind that an event in a stream may be associated with state changes in multiple consuming applications with different requirements. Similarly, the literature on event processing emphasizes that the degree of consistency is highly dependent on the use case and end-user requirements [@barga2006consistent]. 

##### Let the User Decide.

In a one-size-fits-all event store, the user should be able to decide for the consistency model. However, the complexity of such an implementation is beyond the scope of this thesis. We focus on one particular consistency model in the implementation of our demo application and highlight its trade-offs. wAt the same time, we refer to the insights gained in this section to develop such an application with multiple consistency layers that allows the user of the event store to decide on consistency, e.g., on a per-stream basis.

We refer to Gotsman et al. [@gotsman2016cause] to help deciding for a consistency model. If low latency is important, we strongly advice system architects to think about their event design and apply practices of _domain-driven-design_ (DDD), such as breaking down their application to trivial facts (domain events) and derived aggregates and categorizing them as monotonic or non-monotonic, as described in subsection [@sec:consistency-decisions]. Afterwards, the appropriate consistency model can be decided upon, which further guarantees safety and liveness for the adapted system design. If latency is not that important or can be mitigated in other ways through sharding and other practices, we recommend opting for strong consistency, as this ensures safety in any case (if thee are no out-of-order events) and allows starting with a simple system design that can be better thought out afterwards. Regarding out-of-order events, they must be either disallowed to provide strong consistency, or the consistency level must be explicitly flagged as eventual consistency, even if a strongly consistent replication protocol is used. We also recommend to look at time-bound partial consistency, as it can help with out-of-order events (subsection [@sec:time-bound-partial-consistency]). We hope that this work will serve as a guidance here.

##### The Model Decision.

<!--
TODO "why consistency is the better choice for most databases"
https://www.cockroachlabs.com/blog/limits-of-the-cap-theorem/
-->

<!--
\todo{Incorporate this statement}

I believe that maintaining eventual consistency in the application layer is too heavy of a burden for developers. Read-repair code is extremely susceptible to developer error; if and when you make a mistake, faulty read-repairs will introduce irreversible corruption into the database.
-->

Accordingly, we opted for strong consistency as the strongest of the identified models. It allows the event store to act as a platform that suits all use cases at least regarding safety. In addition, this allows us to built for a set of use cases where data safety is a top priority, and enables use cases where patterns that include the non-occurrence of events are to be detected. With strong consistency, we can start with a prototypical implementation that is easy to understand, compared to other protocols. We then evaluate this implementation to understand the trade-offs for latency and availability not only in theory, but also in practice. We are aware that this introduces network latency and decreases the maximum throughput on a single stream, while this effect can be mitigated with horizontal scaling through partitioning and sharding. Many popular distributed databases have proven that extremely high throughput rates are possible even with strong consistency, as we have shown in section [#sec:previous-work]. This serves as a basis for future work, that could result in a distributed ChronicleDB that allows users to choose their consistency level on a per-stream basis, or even across streams. 

We identified a few strategies how to cope with out-of-order events, but we do not decide on and implement a specific one other than a lightweight buffer due to the limited scope of this work.

This considerations justify our decision that we add the following requirement to the distributed event store:

- **Linearizability**: To ensure the safety of the event streams, every write to an event stream will be ordered, i.e. every event is written in the order given by its timestamp, and every query will return the events in this order. An expection to this can be made with out-of-order events, that must be written ordered in insertion time, but still returned ordered in event time.

##### Trade-Offs.

With strong consistency and buffered inserts, we provide a generalized solution that somehow works in every case, but not perfectly in a specific use case. This can only be achieved by allowing users to control various parameters that affect consistency. There are latency and availability trade-offs to be made. This is not necessarily a bad thing: We have learned that compromises cannot be avoided, while the effects of the compromises may not be so bad in practice. We will introduce other strategies to cope with this, such as partitioning.

The buffer can handle out-of-order arrivals of events, if its size and timeout are chosen wisely. But as this will always be a heuristic, events might still arrive out-of-order. In addition, the buffer increases the latency of write acknowledgements, as any event is only written and acknowledged once it's batch is acknowledged. 

### Deciding for a Replication Protocol

Of the protocols presented in detail in chapter [@sec:background], the protocol of our choice is Raft (see subchapter [@sec:raft]). Here follows a list of reasons that justify the use of Raft for our purposes:

- It provides strong consistency,
- An implementation can be derived from the complete and concise specification, which reduces the chance to introduce violations of correctness guarantees,
- It is a understandable protocol, which facilitates the application and customization of the protocol,
- It is commonly used by the largest distributed systems vendors in large-scale production systems (cf. section [@sec:previous-work]),
- Academic research on Raft has been trending in recent years and will likely continue to do so in the coming years, allowing for a reliable stream of updates and improvements to the protocol,
- There are many ready-to-use implementations of Raft with proper APIs written in major programming languages and supported by large communities that will also adopt improvements from recent scientific research,
- Multi-Raft allows us to implement partitioning and sharding natively,
- The monotonically growing append-only write-ahead log that contains the commands (i.e. the events) could be naturally paired with the ChronicleDB event log according to "the log is the database",
- The log also allows to work with out-of-order events efficiently, compared to primary-copy replication,
- In the face of strong consistency, the cost of replication are justifiable in Raft.

<!--
Primary-copy like ZAB would also work, as the diff of states after each operation == event(s) - no big difference to SMR here, but with OOO sending the diff would mean deriving the op again - because sending the insert index of the event and the event itself is not sufficient
-->

### Raft Implementations

\todo{Just the list from the raft github? Or omit}


#### Library Decision

\todo{Rephrase}
- Building it from ground up makes sense if you want full control and adjust the protocol to perfectly fit your use case (TODO find and cite the paper/tool that mentioned that, it must have been mongo or couchbase)
- Adjusting the protocols allows you to tickle out the last ounce of performance, but oftentimes under the cost of losing the formal verification (TODO reference to raft verification mention in 03b)
- This thesis is about a proof that it works, so we use something existant to leverage implementation power of the OOS community and save time to build the proof
- We use Java for simplicity, as ChronicleDB is written in Java. For best performance and future-proof support (at the time of this writing), it is recommended to implement the library yourself or to use one of the popular Golang implementations: etcd/raft or hashicorp/raft. One could even use the Gorum framework in Golang (with some adjustments as in [@pedersen2018analysis])
- It is also possible to build upon a library / an API and also leveraging the library in a way that allows you to maximize throughput
- TODO List other libraries in short

Note: Implementing a replication layer that allows the event store consumer to decide on the consistency level, would in general require to implement the replication protocol yourself. As we have seen so far, there is no out-of-the-box implementation of multiple protocols that can just be toggled in between. We found in Apache IoT DB the approach to toggle the protocol by wrapping it into a provider, but only for raft, multi-leader consensus and standalone usage (https://github.com/apache/iotdb/tree/master/consensus/src/main/java/org/apache/iotdb/consensus). Another approach would to limit the toggleability on a per-stream basis and deploy two completely different replication protocols/libraries.


#### Apache Ratis

\todo{Rephrase}
- Repo, maybe mvn link, version 2.1.0 [@apache2022ratis] [@apacheratis2022github] 

TODO from https://de.slideshare.net/Hadoop_Summit/high-throughput-data-replication-over-raft:
Raft protocol has been successfully used for consistent metadata replication; however, using it for data replication poses unique challenges. Apache Ratis is a RAFT implementation targeted at high throughput data replication problems. 
(Background Info: Has been an incubator project at Apache; was approved; same team (?) from Hortonworks built Apache Ozone on top (validate this!); Hortonworks merged with Cloudera)

° In Raft,
* All transactions and the data are written in the log
* Not suitable for data intensive applications

° In Ratis
* Application could choose to not write all the data to log
* State machine data and log data can be separately managed

* See the FileStore example in ratis-example
* See the ContainerStateMachine as an implementation in Apache Hadoop Ozone.

To do so, must implement interface StateMachine.DataApi:
An optional API for managing data outside the raft log.
For data intensive applications, it can be more efficient to implement this API
in order to support zero buffer coping and a light-weighted raft log.

### ChronicleDB on a Raft

This subsection describes the actual implementation of Raft in ChronicleDB with Apache Ratis.

\todo{Algorithm diagrams}

like in https://software.imdea.org/~gotsman/papers/unistore-atc21.pdf

#### Limitations

On top of the constraints we set for the research, we also limit the scope of our prototype implementation to fit within the bounds of this work. For topics not covered here, see also the section on future work in Chapter [@sec:conclusion].

##### Sharding.

We limit our implementation to the one-partition-per-stream case, thus we are not able to evaluate the impact of sharding on the overall througput.

##### Message Protocol.

We use gRPC with Protocol Buffers for intra-cluster messaging between nodes. But, we currently only offer HTTP/1.1 for clients to communicate with a deployment of ChronicleDB (next to the Java API for the embedded case). We recommend using gRPC (which is built upon HTTP/2), AMQP, or other protocols that are best suited for high-velocity event messaging.

##### Advanced Out-Of-Order Handling.

While we recommend to invest time into researching and implementing an approach to cope with out-of-order events such as punctuation, time-bound partial consistency with inconsistency windows, or a consistency query module, the scope of this work is limited. Hence, our prototype implementation does not provide such a module. Instead, we will implement a lightweight buffering approach for writes that dramatically reduces the likelihood of inconsistencies due to out-of-order events in practice.

#### Technical Architecture Overview

<!-- Now, I present my results -->

- Stack:
    - Protocol Implementation: Apache Ratis
    - Messaging Implementation: Google Protocol Buffers / gRPC (TODO cite both)
    - Event Store: Standalone/Embedded ChronicleDB Event Store + ChronicleEngine

- pretty architecture diagrams

- Architecture in overview: Communication layers, client + server architecture, node architecture, how to run the system, ... afterwards the details in own sections

- Describe target package/library ecosystem (replicated event store as lib, consumed by spring)

two separate groups of processes: data nodes and meta nodes. Exposing 3 ports: Data, Metadata/Management and Public (HTTP/REST). Similar to many other solutions like InfluxDB, Apache IoTDB... (but by accident, not intentional :) )
See https://docs.influxdata.com/enterprise_influxdb/v1.8/concepts/clustering/
https://iotdb.apache.org/UserGuide/Master/Cluster/Cluster-Setup.html 

<!--
#### Simplified API for Apache Ratis

> This should better be covered in the following sections (cluster management + state machine)

- StateMachineProviders
- PartitionInfo
- ClusterManager
-->

\paragraph{Changes in the ChronicleDB core.}

\todo{Mention the other patterns we used other then facades}
Since we wrapped the relevant classes and interfaces of ChronicleDB into own extensions and facades, no changes in the core implementation of ChronicleDB where needed.

We created an interface to the `ChronicleEngine` and extended it to a replicated one, serving as a facade and the API for the end-user. 

We... EventStore...

To allow for distributed queries, the original query interface of ChronicleDB should be adjusted, since it exposes a cursor interface that is hard to wrap in a serialized gRPC command message. Instead of wrapping, it should be re-eingeered for the distributed case. Note that we limit our work to support replicated writes, so we did not touch this yet.

#### Cluster Management

- Similar to LogCabin (see previous work)
- The management quorum
- Bookkeeping of available nodes and balancing partitions
- Heartbeats, health checks (failure detections) and timeouts
- Registering of Partitions for StateMachines
- Additional RaftServer (with own port)
- Explain how to startup the cluster similar to https://github.com/logcabin/logcabin/blob/master/README.md
- Failure detection (TODO also reference 03a)
- TODO chain replication states to Allow to build a distributed system without external cluster management process; raft did it, too (see KIP-500), but why did I ended up in here? Describe this so it makes sense (multi-raft, partitioning, add work item to conclusion to get rid of the cluster manager process)

- TODO all commands of the cluster mngmt SM in mathematical notation, and short listing of protobuf example

#### Failure Detection

\todo{Describe the heartbeat mechanism} 

#### Event Store State Machine

- Standalone Event Store wrapped
- Replicated Event Store as a facade / to be used as the client to the cluster
- TODO think of moving it out of spring boot, use as standalone lib
- TODO if reasonable, list algos

- TODO all commands of the SM in mathematical notation, and short listing of protobuf example

TODO the state machine manages a reference to a copy of the event stream via a EventStore instance


In its core, a very simplified state machine for ChronicleDB could be describes as

$C$ set of all possible events $e$

$S = \{\dots\}$ all possible sequences $E$ of events

$\delta(e, ()) = (e)$ where $()$ is the empty sequence

$\delta(e, (e_1, \dots, e_n)) = (e_1, \dots, e_n, e)$

$\delta'((e'_1, \dots, e'_m), (e_1, \dots, e_n)) = \delta (e'_m, \dots \delta(e'_1, (e_1, \dots, e_n))\dots) = (e_1, \dots, e_n, e'_1, \dots, e'_m)$

TODO as far as the log consists of commands rather than values, we describe it as 

$\delta(cmd(arg), ()) = cmd'(arg, ())$

And concretized

$\delta(append(e), \empty) = (e)$

$\delta(append(e), (e_1, \dots, e_n)) = (e_1, \dots, e_n, e)$

#### Log Implementation

- Commands
- Replayable
- Transactions vs Queries
- Acts kind-of write-ahead log, but buffered
- Snapshot: Pointer to last committed event in the store
- Persisting the log expensive
- TODO compare with other databases write-ahead logs
- OOO challenges
- TODO if reasonable, list algos

\todo{mention here AND in conclusion}
TODO the log is a very naive implementation. Popular applications even use efficient embedded (in-memory?) storage engines such as RocksDB (like CockroachDB https://github.com/cockroachdb/cockroach/issues/38322) - and they also come with WAL logs, so we have a multi-layer architecture of the raft log. Even if this comes with some issues, we can learn from it and use a better log approach. Our naive approach (the Ratis default one) can slow down the system. Makes sense to have the raft log running on a different thread to not block other operations and improve I/O

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/chronicle-raft-log.pdf}
  \caption[Relation of the Raft log and the event store]{Relation of the Raft log and the event store. a) Appending events without allowing for out-of-order events always results in the same sequential order of the events in the Raft log (wrapped in operations) and the events in the event store, even if insertion and event time differ. b) With buffered inserts and without out-of-order insertions, the order correlation still applies, as the buffer contents are ordered before insertion and the buffer can only be served from a single leader node. c) As soon as out-of-order insertions are allowed, the order of the two logs given by insertion and event time is no longer guaranteed to correlate. This can break previously derived non-monotonic aggregates, as shown in figure \ref{fig:ooo-consistency}.}
  \label{fig:chronicle-raft-log}
\end{figure}

##### Log-Less Rafting.

With the Raft log, we write the same event twice: Once to the Raft log, and once to the event stream (when it is replicated and committed). The messages end up being stored redundantly, and the duplicate writes use up additional I/O capacity, which slows down the system. It also violates the _"The log is the database"_ philosophy of ChronicleDB. So the question is: why can't the event stream simply act as the log, instead of a redundant Raft log?

- The Raft log contains not only events (actually insert commands), but also Raft commands (such as leader changes, network reconfigurations etc.) and in the future additional operations on the event stream (for example, maybe for schema evolution, but here we would advice to just create a new stream).
- With out-of-order events, the ordering of the Raft log and event stream can differ (see the previous discussions in section [@consistency-choice]).

We can address this problem if we reduce the raft log to pointers, at least for the write operations. We use the event stream as the truth for the log and keep only pointers in the Raft log (next to the command, in this case `insert`). Once a message arrives at a node, it immediately writes it into the event stream and at the same time a pointer to the Raft log. When reading the event stream, the system must read the high-water mark of the log to know which event stream entries are already committed, and is only allowed to return these. In case of uncommitted entries dropped from the log (e.g., after a network partitioning or due to some faults), the event stream entries after the high watermark are allowed to be overwritten. Overwriting entries is a new concept to ChronicleDB that must also be discussed from an index and storage layout perspective, therefore we haven't implemented this throughout this work. This approach must also take care of out-of-order events: it is not sufficient to read the high-water mark and allow to return all past entries in the event stream that have an event timestamp before the watermark, because this could return out-of-order entries that were not yet fully replicated to a quorum and committed. It is only allowed to return entries with an insertion timestamp before the entry with the high-water mark, which requires further thoughtful engineering to be efficient. In addition, no materialized aggregates are allowed to be returned that are based on uncommitted out-of-order entries. This is illustrated in figure \ref{fig:logless-raft}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/logless-raft.pdf}
  \caption[Almost logless Raft]{Sketch of an almost logless Raft. a) Regular Raft log, with events only written to the stream once committed, but then, they are written twice. b) Almost logless Raft, where the Raft log only contains pointers to the actual events in the stream. Events are immediately written to the stream, but only those up to the high-water mark are allowed to be read by clients. c) This simple approach won't work if out-of-order events come into play.}
  \label{fig:logless-raft}
\end{figure}

Ignoring out-of-order events, snapshotting the state and compacting the log is also easy, because it will mean no more than just removing old pointers that we are not interested in and keeping the high-water mark as the pointer to the most recent stream event.

There are also more approaches to logless state machine replication in the literature, that goes way beyond this. So far, they are only suited to smaller key-value stores, as they manage a full sequence of states, but research here could be deepened for larger structures such as an event store [@skrzypzcak2020towards].

#### Buffered Inserts

Cf [@sec:cost-reduction]

From original diss file:///Users/christian.konrad/Documents/Paper/style%20inspiration/OngaroPhD.pdf:

"Raft supports batching and pipelining of log entries, and both are important for best performance.
Many of the costs of request processing are amortized when multiple requests are collected into a
batch. For example, it is much faster to send two entries over the network in one packet than in two
separate packets, or to write two entries to disk at once. Thus, large batches optimize throughput
and are useful when the system is under heavy load. "

Unfortunately, ratis does not support batching itself. Therefore, we implemented a buffer ourselves to provide batching.
(TODO mention that its better to implement batching in raft, so log entries stay atomic and do not bundle multiple operations per log entry like in our case)

It is recommended to insert events in batches to minimize the network overhead (cf. InfluxDB)...
To make this easy for the user... don't move batching into the responsibility of the client or the user... ChronicleDB can be run with buffered inserts (optionally)... strong leader buffer (not implemented yet)

TODO buffer also allows for concurrency, as multiple threads can write to the buffer, while its content is ordered on flush. Mitigates overlapping writes, as buffer content is always ordered by event time.

TODO buffer must be on a single leader, otherwhise order of the batches can not be ensured (would need a merge layer for batch writes...)

and also improve overall latency using buffer... cf. ring buffer in cockroach

https://docs.influxdata.com/influxdb/v2.3/write-data/best-practices/optimize-writes/#batch-writes


TODO log is sequentially ordered by time of imsert... using java concurrency mechanisms... what if communicated via rest? not linearized
TODO buffer: fire and forget. wo/ buffer + sync: strong linearizable
TODO buffer violates strong consistency: Events are only ordered in the buffer, and the event sets of the buffers are ordered via raft log, but when applying to state machines, the events of the buffers can overlap. Events are still ordered in the event store, but inserting them needs a lot of OOO.
TODO for future work: Only allow the leader to apply to its buffer. We didn't managed to do this in this work due to time issues, but this is should be the final architecture to ensure strong consistency

... buffer allows for concurrent appends... multihreaded writing and reading from the buffer... buffer flushed when full or after timeout... but must also follow single-leader practices (see above)

TODO when not applying single-leader characteristics to the buffer, the buffer would dramatically increase the chance for bulky out-of-order inserts that are too large for the spare space in the blocks, resulting in a hard slowdown... We need to enforce single-leader to the buffer. (Unfortunetally, the current experimental implementation does not enforce single-leader to the buffer atm). But with single-leader, the buffer reduces chance of OOO because it pre-sorts before flushing. 

\todo{Move to general architecture overview}
Two modes: Async fire-and-forget and synchronous. The latter is important when a client needs the guarantee that one event is written before it continues sending futher events. The async mode allows for very high throughput. With horizontal scaling, it is possible to keep a constantly high throughput rates (events/s). With higher replication factor, it decreases (TODO how does Zeebe keep throughput constant when latency increases???). Thanks to the buffer, intermediate higher event emitting rates can be compensated, if the mean rate stays below the max throughput... Otherwhise, the overall system slows down and probably crashes ATM (future work: Monitoring of the system, truely elastic, but due to append-only and linearizability we can not mitigate everything with partitioning if write rates stay too high for a minimum replica set; there will be a upper bound. Same would apply to eventual consistency btw: If mean write rate stays higher than max throughput, the system will neve become consistent (i.e., replica states will never converge (TODO should we mention that in fundamentals/cost of replication?)))

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/ha-chronicle-architecture.pdf}
  \caption{Architecture of a ChronicleDB cluster for high availability}
  \label{fig:ha-chronicle-architecture}
\end{figure}

Figure \ref{fig:ha-chronicle-architecture} illustrates the architecture:

\todo{Grammar check}
\begin{enumerate}[label=(\arabic*)]
  \item A client emits events to be inserted into the store. This happens either through a remote API (in this case, a REST API) or directly on a one of the nodes via the Java API for an embedded design.
  \item The partition manager of the ChronicleDB engine finds the right partition for the insertion request and the current leader for this replica group to serve the request. The partition and leader mapping is cached for a performanant in-memory lookup. The request is sent to this leader, while the client is notified for direct access of the leader for subsequent requests.
  \item The inserts are added to a non-blocking buffer that allows for synchronized concurrent inserts and writes. The buffer is flushed once it overflows, at least after a certain timeout after the last insert. The buffer is optional and both its maximum size and timeout are configurable.
  \item The flushed buffer content is inserted into the event store in a batch operation. Before inserting, the events are ordered in-place in the buffer to prevent out-of-order inserts in the given set. The flush and insertion happen in a single, atomic operation.
  \item The insert of the events happens in a batch. The insert operation, including the event payloads, is appended to the raft log, replicated and applied to the state machine. Following the Raft consensus protocol, the insertion is acknowledged once a majority of the nodes committed the operation. The state machine manages a state object, that itself encapsulates the actual ChronicleDB event store index.
\end{enumerate}
 
\todo{Is it true that flush and insert is one atomic operation?}

TODO sketch the raft log (one log entry is one batch insert operation (or one create aggregate op))

TODO log holds kind of transactions?
"Standard database systems are not designed for supporting such write-intensive workloads. Their separation of data and transaction logs generally incurs high overheads."

--> Compared to RDBMS, not near the overhead of real transaction logs (including locks etc)

"Data loss due to system failures or system overload is generally not acceptable" <-- Buffer can fail and causes loss of its data - TODO mention in future work that we should have somehow persistence of the buffer or use another type of buffer 

\todo{Describe the state machine and the state manager}


TODO list this possible improvement ideas for our buffer
TODO mention that in practice, people use a ring buffer
TODO ring buffer??? 
- https://groups.google.com/g/lmax-disruptor/c/lWqnUaclPIM
- https://github.com/yburke94/Raft.net "This framework makes heavy use of ring buffers to facilitate message passing between threads."

"When a command is executed against the cluster leader, it is placed in the leader ring buffer. This buffer has 4 consumer threads that perform the following for each entry in the ring buffer (in order):

Encode the log entry (done using ProtBuf).
Writing the log entry to disk (using the journaler).
Replicating the log entry to peer nodes.
Updating the internal state of the node."

also https://github.com/hashicorp/raft/blob/main/log_cache.go 
"// LogCache wraps any LogStore implementation to provide an
// in-memory ring buffer. This is used to cache access to
// the recently written entries. For implementations that do not
// cache themselves, this can provide a substantial boost by
// avoiding disk I/O on recent entries."

Also https://blog.bernd-ruecker.com/how-we-built-a-highly-scalable-distributed-state-machine-f2595e3c0422

"This also allows you to use multiple threads without violating the single writer principle described above. So whenever you send an event to Zeebe, the receiver will add the data to a buffer. From there another thread will actually take over and process the data. Another buffer is used for bytes that need to be written to disk."

- TODO pretty architecture diagram (buffer state, timeout, flush... + what's in the raft log)
- TODO reasoning, performance and concurrency considerations
- TODO compare the 2 buffer approaches (blocking vs. non-blocking)
- TODO explain impact of buffer sizes
- TODO explain similarities with Raft Log Buffer
- TODO if reasonable, list algos
- TODO the buffer considerations must be made based on the RPO: The buffer timeout must be smaller than the RPO
- TODO as the buffer holds data temporarily on a single node (leader), if the leader crashes, this data is lost (so it's not strong consistent)

<!--
- What if multiple clients (= devices) write to the same schema?
-> Easy solution: We could use one stream per client/device
- If we go for a single stream: We are linearized regarding ingestion time (but not for overlapping requests, which is the characteristic here)
- But not for event time (no strict consistency), so we can expect a lot of OOO
- But: The buffer also helps here to avoid OOO. Strict concistency is too expensive: Assuming the clocks of all devices are the same (which is rare; we only have that with infra in full control (cf. Google example)), we would still need to order events before storing in the store, but we won't know if there are still events to come (due to network latency of some clients). This would require extremely expensive coordination between clients. This is only of theoretical interest: we don't need that in practice due to OOO handling and the buffer
-->


##### Limitations.

As we pointed out in subsection [@sec:buffer-theory], we should acknowledging each write to the producer by keeping a callback reference for each write. This is challening and we haven't implemented that in the prototype. At present, only a successful write to the buffer is confirmed, but this does not guarantee a successful write to the quorum.

#### Partitioning using Multi-Raft Groups

- Allows for scaling
- The throughput of a single event store instance has an upper bound (based on I/O capabilities of the underlying machine, the OS, concurrent access to the ressources of that machine, the implementation of ChronicleDBs index and last, usually with the most impact, the replication protocol and the underlying network)

cf. [@sec:partitioning]
- Basic vertical partitioning by stream, no sharding
- One raft group = one partition containing an instance of a certain state machine, distributed over n nodes (n = replication factor)
- PriorityBlockingQueue for simple load balancing
    - Naive implementation (balanced per absolute # of partitions, not by time splits or per actual load); better partitioning approaches see background#Partitioning and Sharding + conclusion
- Show diagram of partitioning approach
- "data partitions can be defined according to both time slice and time-series ID" [@wang2020iotdb]
- In this implementation, currently only per stream ID
- Raft built-in network reconfig allows for rebalancing (cf. [@sec:partitioning]) without compromising availability

But no real sharding: Events belonging to a single schema are not not split in the current solution.
But: Consider time splits in future work

when rebalancing, replicas may be moved across nodes.  TODO show my figures on rebalancing

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/multi-raft-architecture.pdf}
  \caption{Partitioning and replication with multiple raft groups}
  \label{fig:multi-raft-architecture}
\end{figure}


partition = raft group on a node (in Ratis, called a division?)

\paragraph{Time Splits.}

In figure xxx, $A_{t_1}^L$ denotes a... when we restrict OOO for historic time splits, we could also replicate them leader-less, as there won't be any writes (TODO aggregate indexes on older splits)... and read from every available node... 

Time splits allow for read scaling; but for chronicleDB we need some strategies to execute queries and aggregates across time splits. (TODO describe strategies; TODO image showing query range |---| with one full and one half split; full split will just moved into memory (= the root of the index tree) while for the half split, the right leaf of the tree needs to be found)


TODO describe the benefit in read scaling with time splits (while writes on the same stream won't scale, as all of them (if we restrict OOO on historic splits) happen on the most recent split) with numbers if possible

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-load-balancing.pdf}
  \caption[Load-balanced partitioning using time splits]{Partitioning using time splits with a replication factor of 3, load balanced on 5 nodes. Metadata is replicated on every node (at least on every node in the bootstrap list) to ensure cross-cluster disaster resilience and fault tolerance and allows the partitions on the respective nodes to directly read the metadata on its own node.}
  \label{fig:multi-raft-load-balancing}
\end{figure}

( TODO is this extra hard replication on metadata neccessary? -> may describe possible byzantine tolerance. But what if you scale with hundreds of nodes? -> discuss this in conclusion)

\paragraph{Sharding (Write Splits).}

TODO also mention hash split keys to have write-scaling.
This allows to scale with the number of writing clients. 
Once a historic time split can be created (and a new set of write splits), the current split can be merged in the background by another job (if current latency allows for it) or just marked as historic splits.
The write splits allow for faster writes, but when querying or aggregating, right strategies to resolve the operation across the shards need to be found. Every shard only offers sequential consistent data; they need to be merged to return the actual linearized stream. But thanks to the sequential ordering of each shard, we can just use in-memory merging, e.g. with a min-heap in $\mathcal{O}(N k \log k)$ time and $\mathcal{O}(Nk)$ space complexity, where $N$ is the number of events per shard (in case of balanced shards) and $k$ the number of shards (TODO reference)... which benefits from smaller time splits and also tumbling windows) and intelligent caching (TODO is there a better merge strategy for reads? The problem is: We write while we read; but if we exclude OOO we can guarantee that a cached read range won't change! problems only occur with OOO) 

TODO as shown by other event store vendors, sharding increases the throughput linearly

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-historic-splits-sharding.pdf}
  \caption[Multi-raft cluster with historic time splits and sharding]{A multi-raft cluster with historic time splits and sharding of the current time split to improve write throughput and latency}
  \label{fig:multi-raft-historic-splits-sharding}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/event-stream-merging.pdf}
  \caption[Two event stream shards merged on a query]{Illustration of two event stream shards merged on a query with tumbling windows}
  \label{fig:event-stream-merging}
\end{figure}

TODO illutrate query over historic trees

|--- leaf --- root --- root --- |

<!-- tumbling windows 

https://ordina-jworks.github.io/kafka/2018/10/23/kafka-stream-introduction.html

-->

<!-- See for discussions around shard capacity and max throughput https://pt.slideshare.net/frodriguezolivera/aws-kinesis-streams?next_slideshow=true -->


TODO glossary with new words in the margin

##### Consistency Across Partitions.

With our partitioning approach, we do not provide strong consistency across streams, since we do not ensure that the order of events to be written into each event stream matches their insertion time. This is fine, since we are only interested in the actual event time. If there is causality between streams, this only becomes relevant when querying across multiple such streams and should be handled by the distributed query processor. We haven't implemented this in the context of this work.

For the sharding case, it looks as follows: Even though sharding (in combination with horizontal scalability) significantly increases overall throughput, it causes a stream to lose consistency without proper inter-shard coordination, which increases latency and thus decreases throughput again. How this could look like architecture-wise is illustrated in figure \ref{fig:chronicle-consumer-producer}. We also described this in the sense of causal consistency in subsection [@sec:stream-causal-consistency].

Since we do not support sharding of a single stream currently in the demo application, we do not need to care about cross-shard consistency (i.e., guaranteeing events are written in event order to the shards). Across streams, we can not guarantee that either, since each stream is handled by its own partition, thus by its own Raft group. We limit our implementation to the one-partition-per-stream case, thus we are not able to evaluate the impact of sharding on the overall througput.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/chronicle-consumer-producer.pdf}
  \caption[Relation between producers and consumers in ChronicleDB]{Schematic illustration of the relation between producers and consumers, and how consistency across shards may look like. The merge step is also illustrated in figure \ref{fig:event-stream-merging}.}
  \label{fig:chronicle-consumer-producer}
\end{figure}

##### Truly Multi-Raft

"As the number of topics increases, so do the number of Raft groups, each with their own leaders and heartbeats. Unless we constrain the Raft group participants or the number of topics, this creates an explosion of network traffic between nodes."

TODO from https://bravenewgeek.com/building-a-distributed-log-from-scratch-part-2-data-replication/

"There are a couple ways we can go about addressing this. One option is to run a fixed number of Raft groups and use a consistent hash to map a topic to a group. This can work well if we know roughly the number of topics beforehand since we can size the number of Raft groups accordingly. If you expect only 10 topics, running 10 Raft groups is probably reasonable. But if you expect 10,000 topics, you probably don’t want 10,000 Raft groups."

"Another option is to run an entire node’s worth of topics as a single group using a layer on top of Raft. This is what CockroachDB does to scale Raft in proportion to the number of key ranges using a layer on top of Raft they call MultiRaft. This requires some cooperation from the Raft implementation, so it’s a bit more involved than the partitioning technique but eschews the repartitioning problem and redundant heartbeating."

TODO problem also accounts for sharding. So, sharding is only feasible when done truly on multiple machines

##### Load Balancing

\todo{Describe my basic approach of balancing partitions}

##### Rebalancing

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-follower-fails.pdf}
  \caption[Rebalancing in the event of a follower failure]{Rebalancing in the event of a node failure that leads to a raft group with too few replicas, while the leader stays intact. a) A fault makes a node crash (fail-stop). The meta quorum detects this crash and notifies the leader of this group. b) The leader requests a membership change from the meta quorum. The quorum selects a node based on balancing rules and triggers a network reconfiguration. The node is added as a new follower to the group. The leader then installs the current state snapshot on this new follower replica.}
  \label{fig:multi-raft-follower-fails}
\end{figure}

TODO leader fails

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-leader-fails.pdf}
  \caption[Rebalancing in the event of a leader failure]{Rebalancing in the event of a leader failure. a) As the followers no longer receive heartbeats from the leader of their group, they start a vote. The winner of the vote becomes the new leader of this group. b) The cluster manager can now assign a new follower similar to the case in figure \ref{fig:multi-raft-follower-fails}.}
  \label{fig:multi-raft-leader-fails}
\end{figure}

TODO reference raft voting mechanism here again

TODO rebalancing when node comes back

#### Fail-Over Design

TODO Network reconfiguration, together with chronicleDB failover

#### Messaging between Raft Nodes

- using gRPC and Protocol Buffers
- Serialisation
- Every command must be functional, serializable and side-effect free
- Commands wrap event inserts

- Describe briefly the messaging format
- ExecutableMessages with Executors framework
- 1:1 protobuf message per java message wrapper
- protobuf: the message. Java: How to apply the message's command on the state machine's state
- Makes it scalable, modular, message-based instead of monolithic state machines
    - If suitable depends on requirements. If state machine operations should be encapsuled and a closed set, just wrap a StateManager around it and reference it in the MessageExecutors
- High potential for model based programming / code generation (could generate executors and java message objects from proto+annotations)

#### Edge-Cloud Design

TODO what is edge cloud? [@cao2020overview]

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/chronicle-edge-architecture.pdf}
  \caption[Architecture of ChronicleDB in an edge-cloud setting]{Potential architecture of running ChronicleDB in an edge-cloud setting, next to and in combination with stream processing systems. It can act both to ingest and serve data before processing, as well as a data lake after processing. Different applications can query the ChronicleDB event store, either ad-hoc or continously. Both real-time and non-real-time systems can read streams from ChronicleDB, which can result in different consistency levels required.}
  \label{fig:chronicle-edge-architecture}
\end{figure}

\todo{Reference to fundamentals}

TODO draw edge-computing diagram for chronicleDB

\todo{Rephrase}
- This implementation is designed to fit three deployment models, one for each layer of the edge computing architecture [@cao2020overview]: 
1. Embedded event store to run as a library tightly integrated in the application code with highly efficient file-based storage on edge appliance on the _terminal layer_ (= original embedded event store/index)  
2. Standalone event store on industrial PC (= original standalone ChronicleEngine) with EPAs... for the _edge layer_
3. Distributed event store in raft cluster mode for the _cloud layer_

- ChronicleDB is originally designed and optimized to run on the terminal layer:
"centralized storage system for cheap disks running as an embedded storage solution within a system (e.g., a self-driving car)" etc.

- It therefore has both a standalone, embedded version, i.e. to be deployed directly on the IoT devices to be lightweight and ultra-fast, and the replicated solution to be deployed in the cloud for high-availability, fault-tolerance etc etc that can scale with the number of devices / streams
- What is currenlty missing is the sync between the embedded and the cloud one (TODO is this ROWA or Primary-Copy?); a good approach would be to have a Event Processing Agent (EPA, TODO reference) in the sense of Complex Event Processing (CEP, [@buchmann2009complex]) in between to aggregate the raw events to useful ones (the output of the sensors ), maybe built with Apache Spark, Kafka, RabbitMQ...

"ChronicleDB supports an embedded as well as a network mode"

"there are many embedded systems where scalable distributed storage is not the outright solution. For example, in [16], virtual machines of a physical server are monitored within a central monitoring virtual machine, which does not allow a distributed storage solution due to security reasons. Other examples are self-driving cars and airplanes that need to manage huge data rates within a local system."
--> But they are all at least locally replicated... otherwhise, not fault and absolutely not byzantine tolerant, which is a big safety issue in these systems!!!

### Test Application

To test the implementation of the replicated event store and the middlewares supporting it, a test application is needed. In the context of this thesis, a synthetic application is constructed to perform trade-off studies. The application is composed of...

- Spring Boot
- REST, facades...

#### Running in Docker Containers

- TODO simple cluster start with docker-compose

<!--

#### Running on Kubernetes

- TODO only cover it in this section if I get a working prototype running
- Helm Charts

-->

#### End-User APIs

> This may be covered in previous sections

**Java API**: 
- Replicated Chronicle Engine
- Buffer Settings

**HTTP/REST**:
- Lorem ipsum

**User Interface**:
> Only brief description with a few screenshots

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Implementation Summary

- TODO short summary section to start a discussion

<!--
TODOS:

- Write readme.md
- Write unit + integration tests
-->

\pagebreak

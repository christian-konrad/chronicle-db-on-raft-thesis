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

### Challenges

\epigraph{\hfill Problems are not stop signs, they are guidelines.}{--- \textup{Robert H. Schuller}}

The biggest challenge in this work was to find an appropriate trade-off between latency, availability and consistency, and therefore to find an appropriate consistency model and based on this, finding a replication protocol that satisfies the traded-off requirements. In addition, we tried to find proper ways to soften the consequences of the compromises. The trade-offs could only be made after an investigation of the potentially targeted use cases of a distributed ChronicleDB and a general discussion of consistency in event stores (subsection [@sec:consistency-choice]). In addition, handling of out-of-order events is challenging, especially in the face of consistency. It violates consistency, but not from the point of view of the event store, but rather from the point of view of client applications that expect a reliable stream of events. Some replication protocols require a write-ahead-log (WAL), which violates the "The log is the database" philosophy of ChronicleDB.

Throughout the next sections, we will show how we tackled these challenges, and in chapter [@sec:conclusion], we will show how these could be further mitigated in the future. But first, we limit the scope of this work to the part of the challenges that we tried to solve in the following subsection.

### Limitations {#sec:system-design-limitations}

Due to the complexity of this topic, the research and implementation in this work is limited to a few well-justified considerations.

\paragraph{Geo-Replication.}

We limit our work to intra-cluster replication. See the conclusion for some hints for further investigations.

\paragraph{Byzantine-Fault Tolerance.}

We haven't implemented or designed a byzantine-fault tolerant version of ChronicleDB, as this work is limited to a problem scope where byzantine faults are rare (but not yet excluded).

\paragraph{Transaction Handling.}

So far, ChronicleDB does not support transactions, which is actually rare for event stores in general. This is good for us, because we don't need to think about linearizing transactions and making them atomic, which also increases the latency of such a system tremendously. If transactions are ever needed, especially across partitions, we refer to subsections [@sec:partitioning] and [@sec:coordination-free-replication].

\paragraph{Distributed Queries.}

The following implementation in this work focuses on write replication. To query an event stream with shards on different nodes, distributed queries must be performed, which is an entirely separate, complex area of interest that exceeds the scope of this work. We refer here to the literature of distributed queries and implementations in popular distributed database systems [@yu1984distributed; @kossmann2000state; @azhir2022join]. In this work, we have not yet implemented the query interface, even for single-shard queries.

\paragraph{Multi-Threaded Event Queue Workers and Partitioning.}

We don't know how the decoupling of local event queue threads by introducing partitioning affects the possible performance that can be achieved with multiple streams through ChronicleDBs own event queue and worker architecture, as with partitioning, the relation is no longer one-queue-per-stream, but one-partition-per-stream. As we move the actual event store implementation behind the state machine and the Raft log, all inserts are linearized, therefore we don't benefit from implementations of the index that are optimized for concurrency.

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
- **Safety**: Nodes of the distributed event store will never return a false stream of events or false query results.
- **Maintainability**: In case of a failure, it is possible to recover from this failure in a short period of time and without compromising ongoing safety.
- **Integrity**: What has been written can not be changed.
- **Liveness**: Every write to the event store will eventually be persisted.
- **Partition-Tolerance**: Even in the event of network partitioning, the event store is available, and safety is never compromised.
- **Edge-Cloud Readiness**: The event store can be run on all three edge-cloud layers: embedded in another application, standalone on a single node or in a distributed setting, replicated and partitioned on multiple nodes.

Unfortunately, an ideal trade-off free solution can't be achieved. So, what trade-offs are acceptable for us? The trade-offs are described by the consistency model we decide for, which we will look at in the next subsection.

### Deciding for a Consistency Model {#sec:consistency-choice}

In this subsection, we elaborate our consistency model decision in detail, based on the previously listed requirements and use cases of a distributed event store, but also on industry best practices as we presented in section [@sec:previous-work]. Subsequently, we will mark the trade-offs to be made. We investigate the concept of consistency from the perspective of real-time and non-real-time event stream consumers and reveal new fundamental properties of consistency in this context.

<!-- Elaborate in this section in detail why we want strong consistency, also with examples. Later repeat the upper dependability requirements and mark the tradeoffs. -->

Throughout this discussion, we will use the notation and definitions from section [@sec:events], plus the following additional definitions:

\begin{definition}[Event Time]
The event time describes the time that an event occured, in contrast to the insertion time, that denotes the time an event was inserted into an event stream. We will denote timestamps in event time with $t$.
\end{definition}

\begin{definition}[Read Time]
The read time describes the real time of a client that reads an event stream. We denote timestamps in read time with $r$.
\end{definition}

Furthermore, we use the following notation:

- $t$ reflects a single timestamp,
- $e_t \in E$ if the event $e_t$ is included in the event stream $E$,
- $E = \langle \dots, e_1 \rangle \xrightarrow{\mathtt{insert}(e_2)} \langle \dots, e_1, e_2 \rangle = E'$ describes a state transition of an event stream.

#### Consistency Perspectives

An event store introduces two different perspectives on consistency:

- The data perspective,
- The client perspective.

While the replication protocol could ensure strong consistency from the data perspective, the system could still look inconsistent from the client pespective due to the allowance of out-of-order events.

This is because strong consistency from the data perspective only cares about ordering of operations, therefore ordering of `insert` commands, which means it provides a strongly consistent ordering for all replicas by insertion time.

However, in many use cases, clients care about event time. We pay special attention to this case in the further discussion. Figure \ref{fig:event-store-consistency-levels} shows the different levels of consistency that can be achieved with strong consistency and allowance for out-of-order events. Figure \ref{fig:event-store-consistency-perspectives} illustrates the two perspectives in the event of out-of-order events arriving.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/event-store-consistency-levels.pdf}
  \caption[Consistency levels for event streams]{Consistency levels for event streams. a) With eventual consistency, no insertion order is guaranteed at all across replicas. b) With strong consistency on the data level, each replica shows the same consistent insertion ordering, but because of out-of-order events, it can still be inconsistent from the client perspective. c) Without out-of-order, strong consistency on the data level provides strong consistency also on the client level, as it guarantees that ordering by insertion time $\equiv$ ordering by event time.}
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

In general, we distinguish between _real-time_ and _non-real-time applications_ when it comes to stream and event processing[^koerber-offline-online-processing]. This also introduces two seperate client consistency perspectives: the real-time client perspective and the non-real-time client perspective. A stream that appears consistent to a non-real-time consumer may not appear consistent to a real-time consumer. Commonly applied system architecture approaches provide both real-time and non-real-time processing to users and clients. Two popular architectures that provide this approach are the _Kappa architecture_ and the _Lambda architecture_. Both architectures allow for real-time and historical analytics in a single setup. The Lambda and Kappa architectures require that event processing reflects the latest state in both batch and real-time streaming layers (called the _speed layer_ in Lambda), hence requiring consistency. A merging layer allows to query the system for both historical and real-time data at the same time. At the heart of these architectures is an append-only data source—such as ChronicleDB, since it allows for both continous and ad-hoc queries [@marz2011lambda][^lambda-note] [@lin2017lambda].

[^koerber-offline-online-processing]: Real-time and non-real-time stream processing are also referred to as _online_ and _offline_ stream processing. We refer the interested reader to the dissertation of Körber on these two types of processing [@koerber2021accelerating].

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

Such systems can be seen in a way that the stream "moves", while the consuming application stays fixed. With the stream moving through the consumer, it sees all recently appended events in near-real-time, at least with some delay. This is shown in figure \ref{fig:realtime-vs-nonrealtime-consumers} a). Usually, the stream is not scanned event by event, but in time windows, as illustrated in figure \ref{fig:windowed-consumers}.

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

Punctuation works similar to punctuation in written language and marks the end of a substream (analogous to the full stop marking the end of a sentence). This allows us to view an infinite, unbounded event stream as a mixture of finite streams [@tucker2003exploiting]. Inside of such a sequence, order does not matter. No further event violate previously received punctuation. OOO is allowed to happen in such a "sentence", but not across sentences. Punctuation provides a guarantee to a stream that all events arriving after the punctuation will have an event timestamp greater than that of the punctuation.

With punctuations, only the punctuations must be ordered, and events between the punctuations are allowed to be unordered, but must be the same set of events (no out-of-order events are allowed to come after their punctation). To identify the correct set of a sequence, the punctuation message `punctuate({e_1, e_2, \dots})` contains the ids of those events to expect in the sequence. This is similar to the "weak consistency" approach, where only expliticitly synchronized parts must appear in order, and everything in between can be unordered. This means that for all events $e$ with $e.t \geq p_1.t \wedge e.t < p_2.t$, $p_1 \prec e \prec p_2$, while any order between events is valid, where $p_1$ and $p_2$ are the punctuations around $e$.

<!-- Another approach that extends punctuation are _heartbeats_ [@srivastava2004flexible]. -->

In ChronicleDB, a similar approach could be adopted to mitigate the out-of-order problem. With this approach, events would not be required to be inserted in event order, while the punctuation allows for strong ordering guarantees in the aftermath. It would also require event producers to support punctuation. There is literature covering the case that event producers can not produce punctuations themselves, which requires the stream processor or event store to deduce these themselves (called a _heartbeat_ then) [@srivastava2004flexible]. Punctuation processing could either be implemented right in the replication protocol by introducing it in the log and weaking the consistency guarentees between punctuation, or in the event streams themselves. We haven't figured out how this could be implemented, though (what if punctuations arive themselves out-of-order?).

\todo{Draw chart for the shopping cart case with punctuation}

Not all use cases can be served with OOP, as implementing and applying punctuation is far from trivial and can not be used in every situation. Nethertheless, this approach comes in handy for all real-time applications where no strong consistency is needed, such as calculating moving averages and similar aggregates.

##### Non-Real-Time Applications.

In non-realtime applications, the consumer moves along the event stream, while the event stream stays fixed. The consumer can move in both directions, scanning the stream. This is shown in figure \ref{fig:realtime-vs-nonrealtime-consumers} b).

In such applications, out-of-order events are not a big threat, as in general, they arrived a long time ago in the event store when a query is run on historic data. Even more, the eventual consistency model is oftentimes sufficient since it is only necessary that the write operations converge at some point in time in the event store. 

Examples for non-real-time applications are _Business Intelligence_ (BI), data science, process mining or machine learning. We have drawn more examples in figure \ref{fig:chronicle-edge-architecture}. Also, using an event store as a data lake to store derived events after processing does not require strong consistency. 

Historic queries can be used to detect faulty results from real-time queries due to delayed events, as the results of historic queries have a higher chance to be consistent. To repair such a faulty result, it requires the developer to write compensation logic in case the real-time layer already caused executions based on incomplete or stale data, similar to the _Saga pattern_. But, this is only limited to actually compensable operations. For example, how should a business compensate financial transactions in the real world that are made based on wrong assumptions due to incorrect data?

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

In an event store, causal consistency must be evaluated from a client perspective, since the only write operation is the `insert`, and there are no mutations of existing events. Events can have intra-stream as well as inter-stream causal relationships, and these relationships must not be explicitly modeled between stream [@sharon2009eventcausality].

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

Since this is up to the user, we recommend to design streams in such a way that the causal relationship is kept coherently on a per-stream base. That is, if a stream can be segmented in a way that the segments are no longer causally related, those segments should be streams of their own, and if events appear to be causally related across streams, it may be appropriate to merge the streams, provided their schemas are compatible. This also increases the performance of the system, as `insert` operations of unrelated events do not need to be coordinated. However, some other properties must be taken into account, such as data locality (keep seperate streams close to where they are created), edge-cloud design (merge streams of two sensors afterwards on the edge or cloud if they are causally related) and sharding (find a balance between causality and scalability; i.e. if the throughput limit of one stream has already been reached, it makes sense to partition it even if events in these shards are causally related).

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/causal-across-streams.pdf}
  \caption[Causal relation across event streams]{Two event streams, i.e. for different domains. a) Inter-stream causality between events of the two streams. b) Due to out-of-order events, intermediate states of the streams violate causal consistency.}
  \label{fig:causal-across-streams}
\end{figure}

##### Out-Of-Order Events and Causality.

Out-of-order events cause violations of intermediate states of streams, as the causality chain can be broken. This is illustrated in figure \ref{fig:causal-across-streams} b). Depending on the consuming applications, this can be a serious problem. Figure \ref{fig:ooo-consistency} shows a generalized picture of how non-monotonic aggregates, which are apparently causally dependent on the source events, can be violated by out-of-order events. Such faults introduced by delayed events would require read-repair code, such as applying the SAGA pattern, to be mitigated in the aftermath, which is prone to more errors to be introduced, and even not always possible.

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

\begin{figure}[H]
  \centering
  \includegraphics[width=0.6\textwidth]{images/convergence.pdf}
  \caption[Convergence of a fixed time window]{Looking at the time window at the head of the stream, we see that it converges after a period of inconsistency when events, including out-of-order events, arrive at the window}
  \label{fig:convergence}
\end{figure}

Continously looking at the head of the stream (the most current time window), under continous writes it will never convergence. If the throughput is acceptable, we will observe a steady convergence rate of 0 (if we write in batches, we may see some stairs-shaped curve). We use this observation in the following paragraphs to describe a way to improve write latency and give better consistency guarantees for reads.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.6\textwidth]{images/convergence-head.pdf}
  \caption[Convergence of the head of the stream]{Simplified illustration of the convergence behavior of the head of an event stream. Continously looking at a time window at the head of the stream (i.e. moving it while we move in real-time), we see that the head of the stream never converges until writes to the stream stop. We can identify throughput spikes. If the average convergence rate is 0, the throughput at this stream and node is healthy.}
  \label{fig:convergence-head}
\end{figure}

A continously negative convergence rate is a cause for concern and signaling that the current throughput may be too high for the system to handle, causing entropy between nodes and clients and finally slowing down the system. This could serve as an ideal signal for elastic auto-scaling: When entropy is detected, new nodes can be provisioned and added to the cluster to handle and balance the additional load. But since only the producing client knows the ground truth of events, it must be responsible to detect convergence by comparing what is written and incoming write acknowledgements. This won't work in an asynchronous fire-and-forget mode, though.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.6\textwidth]{images/consistency-function.pdf}
  \caption[Illustration of consistency functions over time windows]{Two illustrative consistency functions over time windows for a high and a low throughput scenario. Because out-of-order events are distributed in an exponential manner across a stream as the chance for a time window to converge increases over time, the consistency increases when we travel back in the stream. The rate of out-of-order events increases with throughput—and also with the level of concurrency.}
  \label{fig:consistency-function}
\end{figure}

\begin{figure}[H]
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

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/inconsistency-window.pdf}
  \caption[Inconsistency window]{Illustration of the inconsistency window. The events in this window are not immediately visible to consuming clients because out-of-order events are likely to appear here.}
  \label{fig:inconsistency-window}
\end{figure}

\begin{definition}[Out-Of-Order Probability]
The out-of-order probability $P_{\mathtt{OOO}}(E_{[t_i, t_j]})$ describes the chance for one out-of-order event to appear in the stream in the time interval $[t_i, t_j]$. We illustrate this in figure \ref{fig:ooo-probability}.
\end{definition}

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/ooo-probability.pdf}
  \caption[Out-of-order probability]{Illustration of the out-of-order probability for the inconsistency window and the remainder of the stream.}
  \label{fig:ooo-probability}
\end{figure}

\begin{definition}[Inconsistency Probability]
The inconsistency probability of a time window describes the probability of that window to be inconsistent, thus $P_{\mathtt{IC}}(E_{[t_i, t_j]}) = 1 - P(C(E_{[t_i, t_j]}))$.

In an event store with strong consistency, this is equal to the out-of-order probability.
\end{definition}

The out-of-order and inconsistency probabilities can be estimated by continously observing an event stream and are served on a per-stream base. In the non-faulty case, the maximum size of the inconsistency window can also be estimated based on communication delays, system load, and the number of servers involved in replication [@lindstrom2014real]. ChronicleDB supports calculation of the _maximum delay_ for events on the fly. It describes the maximum amount of events that are present between an out-of-order event and the place that it was supposed to be inserted at according to application time [@seidemann2019chronicledb]. This allows to derive the inconsistency window with built-in methods.

With this, rather then deciding for the heuristic watermark, a real-time application could agree on a tolerated inconsistency probability. This means that they would define the minimum required consistency probability $p_{C}$ for the remainder of the stream (the sequence of the stream that is allowed to be read) by configuration. This is equal to the the minimum inconsistency probability that the inconsistency window must cover, making it simple to argue about the length of the inconsistency window (and to derive the heuristic watermark):

\todo{If time, denote this as an algo}

We continously observe and register the delays of out-of-order events in the stream and set the window size in a way that it covers the $p_{C}$ percentile of these delays, including an additional tolerance area (such as 20% of this timeframe). We choose the percentile so we can exclude outliers, as shown in figure \ref{fig:ooo-distribution}. We don't recommend to choose a value of 100%, as if any outlier occurs (due to a node fault or other extreme delays), it would render the inconsistency window too long, thus denying any near-real-time properties. 

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
  \includegraphics[width=0.55\textwidth]{images/ooo-distribution.pdf}
  \caption[Distribution of out-of-order events]{Distribution of out-of-order events in an event stream, relative to the current timestamp}
  \label{fig:ooo-distribution}
\end{figure}

##### Guaranteeing the Inconsistency Window.

In theory, the whole stream is still eventually consistent from the clients view, but in practice, inconsistent reads outside the inconsistency window should be very rare. The consistency probability could be turned into a guarantee by covering it in SLAs, for example. Or, we could even harden this property by requiring the following guarantee: All events must be eventually written during the inconsistency window. This would mean that we would reject any out-of-order entry that reaches the event store with a delay larger then the inconsistency window, sending a `nack` to the event producer and making the producer responsible to compensate for the missing event. 

##### Conclusion.

By investigating the convergence behavior of a stream, we can argue about different levels of consistency throughout one stream from the clients perspective. By continously observing convergence and adjusting the inconsistency window on a per-stream basis, we can improve the latency and consistency while providing near-realtime reads to consumers. From a theoretical point of view, this still does not provide a strong consistency guarantee, but the probabilistic guarantees may be sufficient for several real-time use cases where write latency is critical. 

#### Buffered Inserts {#sec:buffer-theory}

One solution to cope with out-of-order events is to buffer `insert` operations temporarily before writing them to the stream. The buffer is flushed and all events written in a batch once its content reaches a certain size or after a certain timeout after the last write. On flush, events in the buffer are ordered by event time, instead of insertion time, ensuring consistency for the batch write. This also decreases latency introduced by the replication protocol dramatically, since the network round-trips are reduced, which we will elaborate later in this chapter. We found similar approaches in the literature [@srivastava2004flexible].

\todo{Refer to this section later again in implementation}

The the buffer size and timeout can be chosen in a way that it covers the max delay of out-of-order events—respectively the inconsistency window. This has a similar effect to implementing the inconsistency window in the query processor, with the difference that events in the buffer are only hold in-memory on a single machine instead of being written directly to the event stream, thus causing the risk of data loss of the events in this buffer. While in our approach, the user must define the buffer size, to guarantee consistency, the buffer size should be adjusted according to _progress requirements_.

The effect of a write buffer on the overall consistency is shown in figure \ref{fig:consistency-function-buffered}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/consistency-function-buffered.pdf}
  \caption[Consistency function using a buffer]{Using a buffer to order events in advance and write them in batches increases the consistency dramatically in the face of out-of-order events and concurrency.}
  \label{fig:consistency-function-buffered}
\end{figure}

While a buffer increases the real-time consistency from the client perspective (the chance is higher that events arrive at consumers ordered by event time), it reduces the data consistency from linearizable to sequential, since after a write, an event can not be immediately read. But, when it is read in real-time, it is ordered (with a high chance). Acknowledging each single write to the stream to the producer is challenging: this would require callbacks that must be managed somewhere, e.g. in the replication log. We currently only acknowledge a successful write to the buffer, but this does not guarantee a successful write to the quorum. Only the acknowledge of a flush of the buffer and the execution of the batch of events lets the client know of a successful write to the store, but this adds a lot of latency to this acknowledgements. In case of unavailability or faults, the client will not get an immediate insight into this.

#### Consistency Across Partitions {#sec:system-consistency-across-partitions}

Keeping writes consistent across partitions, and especially across streams of different schemas, is challenging, especially when each partition is covered by its own instance of the replication protocol. This requires another layer of coordination across partitions, which increases the overall latency. It is also only relevant for streams with inter-stream causality (see subsection [@sec:stream-causal-consistency]).

Instead of linearizing writes across partitions (which anyway only ensures ordering in insertion time), it is better to ensure inter-stream ordering on query time. If there is causality across streams, this becomes only relevant on queries on multiple of such streams, and should be handled by the distributed query processor.

Note that even popular event stores and similar systems often do not support strong consistency guarantees across partitions, such as in Azure Event Hubs [@microsoft2022eventhubsconsistency].

#### Conclusion

What have we decided for? The previous discussion has shown that the consistency model applicable heavily depends on the use case of the event store: event sourcing requires at least causal consistency, ESP relies commonly on strongly consistent data and in CEP, pattern matching also works on monotonicly derived aggregates on eventual consistent data. Non-realtime applications are usually satisfied with eventual consistency. And there are also applications that are satisfied with a strong ordering by insertion time. This shows that the event store is not the application, but a platform, inheriting its consistency requirements from the consumers. We must keep in mind that an event in a stream may be associated with state changes in multiple consuming applications with different requirements. Similarly, the literature on event processing emphasizes that the degree of consistency is highly dependent on the use case and end-user requirements [@barga2006consistent]. 

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

Accordingly, we opted for strong consistency as the strongest of the identified models. It allows the event store to act as a platform that suits all use cases at least regarding safety. In addition, this allows us to built for a set of use cases where data safety is a top priority, and enables use cases where patterns that include the non-occurrence of events are to be detected. New distributed storage systems that we analyzed have proven that very high throughput is still possible in practice when a strongly consistent replication protocol is applied (see subchapter [@sec:previous-work] for examples). With strong consistency, we can start with a prototypical implementation that is easy to understand, compared to other protocols. We then evaluate this implementation to understand the trade-offs for latency and availability not only in theory, but also in practice. We are aware that this introduces network latency and decreases the maximum throughput on a single stream, while this effect can be mitigated with horizontal scaling through partitioning and sharding. Many popular distributed databases have proven that extremely high throughput rates are possible even with strong consistency, as we have shown in section [@sec:previous-work]. This serves as a basis for future work, that could result in a distributed ChronicleDB that allows users to choose their consistency level on a per-stream basis, or even across streams. What we found so far—and this is backed by our observations of other distributed systems—is that metadata should always be distributed with strong consistency, even if data is replicated with a lower consistency level, so that the clients and the cluster nodes themselves never assume an inconsistent cluster state, which can lead to erroneous behavior.

We identified a few strategies how to cope with out-of-order events for those clients with strong ordering requirements by event time. We decided to implement a lightweight buffer to cover the inconsistency window, while we do not implement advanced techniques due to the limited scope of this work. For clients that want to process events by insertion time, linearizability on the data perspective is sufficient to ensure consistency on the client perspective, even if the events occur out-of-order.

This considerations justify our decision that we add the following requirement to the distributed event store:

- **Linearizability**: To ensure the safety of the event streams, every write to an event stream will be ordered, i.e. every event is written in the order given by its timestamp, and every query will return the events in this order. The order can be either by event or insertion time. An expection to this can be made with out-of-order events, that must be written at least ordered in insertion time, but still returned ordered in either event or insertion time.

##### Trade-Offs.

With strong consistency and buffered inserts, we provide a generalized solution that somehow works in every case, but not perfectly in a specific use case. This can only be achieved by allowing users to control various parameters that affect consistency. There are latency and availability trade-offs to be made. This is not necessarily a bad thing: We have learned that compromises cannot be avoided, while the effects of the compromises may not be so bad in practice. We will introduce other strategies to cope with this, such as partitioning.

The buffer can handle out-of-order arrivals of events, if its size and timeout are chosen wisely. But as this will always be a heuristic, events might still arrive out-of-order. In addition, the buffer increases the latency of write acknowledgements, as any event is only written and acknowledged once it's batch is acknowledged. 

### Deciding for a Replication Protocol

Now that we decided for strong consistency, the space of possible replication protocols (presented in detail in chapter [@sec:background]) is narrowed down to a few, such as consensus protocols, state machine replication and derivates. Of these protocols, the protocol of our choice is Raft (see subchapter [@sec:raft]). Here follows a list of reasons that justify the use of Raft for our purposes:

- It provides strong consistency,
- An implementation can be derived from the complete and concise specification, which reduces the chance to introduce violations of correctness guarantees,
- It is a understandable protocol, which facilitates the application and customization of the protocol,
- It is commonly used by some of the largest distributed systems vendors in large-scale production systems (cf. section [@sec:previous-work]),
- Academic research on Raft has been trending in recent years and will likely continue to do so in the coming years, allowing for a reliable stream of updates and improvements to the protocol,
- There are many ready-to-use implementations of Raft with proper APIs written in major programming languages and supported by large communities that will also adopt improvements from recent scientific research,
- Multi-Raft allows us to implement partitioning and sharding natively,
- The monotonically growing append-only write-ahead log that contains the commands (i.e. the events) could be naturally paired with the ChronicleDB event log according to "the log is the database",
- The log also allows to work with out-of-order events efficiently, compared to primary-copy replication,
- In the face of strong consistency, the cost of replication are justifiable in Raft.

### Raft Implementations

There are several Raft implementations in different programming languages. The Raft authors maintain a curated list of these implementations on their website [@raft2022implementations], where they provide an overview of the following properties and features per listing:

- GitHub stars,
- Authors,
- Programming language,
- License,
- Leader election + log replication,
- Persistence,
- Membership changes,
- Log compaction.

There may be a few other libraries that are not on this list, but we haven't investigated that yet because this list looks very comprehensive.

#### Library Decision

Before introducing the library we have chosen, we list the requirements we have for a suitable library implementation.

##### Requirements.

We are looking for a library written in `Java`, since ChronicleDB is written in `Java` and we want to integrate it tightly with the actual event store implementation, rather than offering it as a service alongside ChronicleDB (cf. ZooKeeper).

We are also looking for

- A library with a non copyleft open-source license,
- A library that is likely to be maintained in a future,
- A library that is feature-complete in the sense of the original Raft protocol description: it provides leader election, membership changes, log replication and persistence, handles network reconfigurations, and allows for log compaction,
- A library that looks to fulfill all correctness properties.

##### Apache Ratis.

We went for _Apache Ratis_ [@apache2022ratis] [@apacheratis2022github] since it satisfies all our requirements and more:

- It is written in `Java`, 
- It can be used as an embedded library rather than a dedicated service,
- It is feature-complete,
- It allows to implement both the state machine and the log yourself, but also provides basic implementations for a quick start,
- It uses `gRPC` (`HTTP/2`) as the messaging protocol both under the hood and for replication messaging,
- It is used in a production-ready, high-throughput, data-intense implementation (Apache Ozone [@apache2022ozone]),
- It is offered under a non-copyleft open-source license (Apache 2.0),
- The project is relatively popular on GitHub[^github-stars],
- The team continously works on the library,
- And the project was moved into the Apache Software Foundation after it succeeded in the Apache Incubator.

[^github-stars]: GitHub stars are a good indicator for the popularity of a library and a proxy for the future-safety, as more popular libraries are more likely to be maintained in future.

Ratis comes of course with some caveats:

- The library is not well documented (a large part of the documentation is not only available in English),
- Due to the not-so-good documentation, parts of the API are hard to understand and the learning curve is steep (can best be learned by examining their sample projects, Apache Ozone implementation, and interfaces),
- The backlogs and current project progress are hard to understand from what they list in their JIRA project[^apache-ratis-jira],
- The team does not release updates very often (less than have a year) [^ratis-release],
- There is no (automatic) verification of the Raft correctness properties.

[^apache-ratis-jira]: https://issues.apache.org/jira/projects/RATIS/issues

[^ratis-release]: We are using Ratis in version `2.1.0`; as the time of this writing, there is a version `2.3.0` released.

Unfortunately, we haven't found any `Java` Library with such a correctness verification. We compared its source code to the $\textrm{TLA}^{+}$ specification of Raft and it looks quite valid, while we can not exclude any exceptions in the runtime behavior.

##### Differences to Basic Raft.

The Ratis authors claim that Ratis is not only suitable for metadata replication, but it is actually targeted to high-throughput data replication problems. We have seen in subchapter [@sec:previous-work] that some vendors, such as InfluxDB, do not replicate their data using Raft, but only the meta data, because of the effect it has on the throughput. In theory, their data replication is only eventually consistent, while they claim this is not the case in practice under normal throughput conditions. This is similar to our observations of the inconsistency window (see section [@sec:time-bound-partial-consistency]). With this in mind, Ratis seems to solve a major problem and makes it suitable for our purposes.

Following is how the authors describe their approach [@ratis2018highthroughput]:

While in basic Raft, all transactions and the data are written in the log, which is not suitable for data-intensive applications, in Ratis, applications could choose to not write all the data to log, by allowing engineers to manage state machine data and log data separately.

<!--
* See the FileStore example in ratis-example
* See the ContainerStateMachine as an implementation in Apache Hadoop Ozone.
-->

To do so, they provide an interface `StateMachine.DataApi` for managing data outside the Raft log. This allows to build a very light-weighted Raft log. 

We haven't implemented this yet in our prototype, but sketched out how this could be achieved in paragraph [@sec:log-less].

Ratis also introduces the concept of _divisions_. A division represents the role a node has in a particular Raft group. In Multi-Raft, a node is allowed to take part in multiple Raft groups, thus having multiple different roles at the same time.

##### Alternatives.

Other alternatives are listed as follows and why we didn't choose them.

- **Hazelcast**: Hazelcast is a full-fledged platform for building real-time applications with stream processing capabilities, that also uses Raft under the hood. This sounds very suitable for our case, as this is exactly what we want to build. But Hazelcast is not a embeddable library, but a platform with a magnitude of other features (in-memory key-value store, embedded stream and batch processing, pub/sub messaging). It is available as an open-source edition with community support, but you still build your application on top of the Hazelcast platform, instead of integrating Hazelcast. This introduces additional complexity, especially as we want to build ChronicleDB as a platform itself, which would require a lot of re-engineering of Hazelcast. Hazelcast is therefore aimed at application engineers rather than platform engineers. To embed Hazelcast instead of running it as a platform, you need to register yourself as a commercial partner of the Hazelcast, Inc. company.
- **etcd/raft**: The most popular Raft implementation out there. As it is written in _Go_ (aka. Golang), it is not suitable for our case (only if we build replication in ChronicleDB around a microservice architecture, which is not our goal). We list it here for the sake of completeness. etcd/raft powers etcd, a high-throughput key-value store, which itself is the fundamental basis for Kubernetes.

We use `Java` for simplicity, as ChronicleDB is written in `Java`. For best performance and future-proof support (at the time of this writing), it is recommended to implement the library yourself or to use one of the popular Golang implementations: etcd/raft or hashicorp/raft. One could even use the _Gorum framework_ in Golang—with some adjustments as in [@pedersen2018analysis]. Go is a relatively new language that is very popular in the realm of distributed systems, since it provides a powerful yet easy built-in networking library similar to Erlang, and the language semantics are simple which makes it easier to write, read and maintain distributed algorithms in Go, compared to other languages. Another reason is the already broad support in this area from package providers, which makes it easier to get started.

Another alternative we have considered is to write the replication layer ourselves from scratch. This would have allowed us to implement a replication layer that allows the consumer of the event store to decide on the level of consistency. We have not found a library that allows this, so this must always be implemented by yourself. As we have seen so far, there is no out-of-the-box implementation of multiple protocols that can just be toggled in between. We found the approach in Apache IoT DB to switch the protocol by wrapping it in a provider, but only for Raft, multi-leader consensus and standalone use[^iot-db]. Another approach would be to limit the toggleability to a per-stream basis and deploy two completely different replication protocols or libraries. Due to the sheer complexity of such an undertaking, we quickly dismissed the idea.

Note that in all of the previous implementations we presented in subchapter [@sec:previous-work], the replication layer was built by the engineering teams themselves.

[^iot-db]: https://github.com/apache/iotdb/tree/master/consensus/src/main/java/org/apache/iotdb/consensus

Now that we presented the library we use for our prototype implementation, we will present the implementation itself in the following sections.

### ChronicleDB on a Raft

This subsection describes the actual implementation of Raft in ChronicleDB with Apache Ratis. Before we delve deeper into this, let's look at further limitations.

<!-- TODO Algorithm diagrams like in https://software.imdea.org/~gotsman/papers/unistore-atc21.pdf -->

#### Limitations {#sec:implementation-limitations}

On top of the constraints we set for the research (cf. section [@sec:system-design-limitations]), we also limit the scope of our prototype implementation to fit within the bounds of this work. For topics not covered here, see also the section on future work in Chapter [@sec:conclusion].

##### Sharding.

We limit our implementation to the one-partition-per-stream case, thus we are not able to evaluate the impact of sharding on the overall throughput.

##### Queries.

To allow for any queries, the original query interface of the ChronicleDBs `lambda-engine` must be adjusted, since it exposes a cursor interface that is hard to break down and wrap in a serialized message, which is required to be executed by a Raft node. We limit our work on implementing and investigating writes, since reads are served from a single leader and do not incorporate significantly more network round-trips, thus not being slower than in the standalone deployment—in fact, replication can reduce read latency. This is sufficient for our analysis. Since it was much easier to implement basic aggregate queries, we use them to validate our results. 

##### Message Protocol.

We use `gRPC` with `Protocol Buffers` for intra-cluster messaging between nodes. But, we currently only offer `HTTP/1.1` for clients to communicate with a deployment of ChronicleDB (next to the `Java` API for the embedded case). We recommend using `gRPC` (which is built upon `HTTP/2`), `AMQP`, or other protocols that are best suited for high-velocity event messaging.

##### Advanced Out-Of-Order Handling.

While we recommend to invest time into researching and implementing an approach to cope with out-of-order events such as punctuation, time-bound partial consistency with inconsistency windows, or a consistency query module, the scope of this work is limited. Hence, our prototype implementation does not provide such a module. Instead, we will implement a lightweight buffering approach for writes that dramatically reduces the likelihood of inconsistencies due to out-of-order events in practice.

##### Log Optimization.

I/O on the Raft log can be costly, as all operations must be written to the log before executed on the event store replicas. Without buffering, the log looks almost similar to the actual event stream persisted in the store. In addition, it requires non-trivial state management such as allocation and pruning to prevent unbounded growth.

The log implementation in this work is far from being optimized. Even more, we violate the "The log is the database" philosophy of ChronicleDB. Ratis allows us to implement the state machine and Raft log separately, and allows for a lightweight log implementation to enable high-throughput data replication. There are also approaches for log-less state machine replication in the literature that can lead back to this philosophy [@skrzypzcak2020towards]. But, we limit our work on the replication protocol itself, not on the implementation details of the log. We sketch out a possible log improvement in paragraph [@sec:log-less].

##### Log Compaction.

We haven't implemented a snapshotting mechanism so far. Currently, when the system is rebooted, the whole log is replayed and the $\textrm{TAB}^+$ index rebuilt, which is very expensive. While snapshotting a simple key-value store is simple (which previous literature was limited to), snapshotting the event store by creating clones of it to the current point in time is too expensive, as the store is already written to disk of the replicas. We considered that it is sufficient to reduce the snapshot to a pointer that points to the latest committed event in the store, and changing the state transfer implementation of Ratis to one that transfers the $\textrm{TAB}^+$ up to this pointer. But this method may be limited to an append-only event store—we haven't considered a solution for out-of-order entries yet, nor are we sure that this requires expectional handling at all.

#### Technical Architecture Overview

This subsection outlines the architecture of ChronicleDB on a Raft. The system is implemented in 3 layers, as illustrated in figure \ref{fig:chronicle-raft-layers}:

- **Replication Layer**: This layer is built with Apache Ratis. We use the `Java` API of Apache Ratis to implement a Multi-Raft cluster running a distributed ChronicleDB state machine. This layer is described in subsections [@sec:cluster-management]—[@sec:multi-raft-partitioning].
- **Messaging Layer**: In this layer, messaging between Raft nodes is defined with Google `Protocol Buffers` and sent over `gRPC`. We elaborate the details of this layer in subsection [@sec:messaging]. In addition, there is also messaging between ChronicleDB and clients, which is currently solved with `REST` on `HTTP/1.1` (cf. subsection [@sec:implementation-limitations]).
- **Data Layer**: This is the layer that holds the actual ChronicleDB implementation by wrapping the embedded ChronicleDB `EventStore` implementation and providing an implementation of the `ChronicleEngine` as the API for producers and consumers. We present our changes in this layer in subsection [@sec:chronicledb-core-changes].

\begin{figure}[H]
  \centering
  \includegraphics[width=0.85\textwidth]{images/chronicle-raft-layers.pdf}
  \caption[Layers of the replicated ChronicleDB architecture]{The 3 layer-architecture of ChronicleDB on a Raft. Each nodes runs on a replication layer and communicates with each other over the messaging layer, while they manage multiple data layers (event streams).}
  \label{fig:chronicle-raft-layers}
\end{figure}

#### Changes in the ChronicleDB Core {#sec:chronicledb-core-changes}

\todo{Mention the other patterns we used other then facades}

A few adjustments where to be made on the original implementation of ChronicleDB. Our adjustments are made on the basis of ChronicleDB version `0.2.0-prealpha`. The original implementation consists of multiple dependent packages:

- `chronicledb-event`: The ChronicleDB event store, which is dependent on the following:
  - `chronicledb-index`: The indexing layer of ChronicleDB,
    - `chronicledb-common`: Commons for the ChronicleDB packages,
    - `chronicledb-io`: The I/O layer of ChronicleDB,
  - `event-data-model` from `de.umr.event`: The event meta model, interface and utilities to describe an event schema.
<!-- - `jepc-api` from `de.umr.jepc.v2`: The API for the _Java Event Processing Connectivity_ (JEPC)[^jepc] library. -->

[^jepc]: mathematik.uni-marburg.de/~bhossbach/jepc/index.html

With this setup, ChronicleDB can be deloyed embedded as a library in other `Java` applications.

On top of this exists the `lambda-engine`, which is a Spring Boot app providing both real-time query and batch processing APIs, based on the Lambda architecture. In addition, it allows to deploy ChronicleDB standalone rather then embedded, exposing a `REST` API for clients to insert and query events (and aggregates).

Our prototypical implementation is dependent on the following packages:

- `chronicledb-event` as the ChronicleDB core implementation,
- `event-data-model` as we need the event data model across all layers.

We wrap the embedded ChronicleDB library into the data layer of our implementation and re-engineered parts of the `lambda-engine` to serve our implementation in a test application (cf. section [@sec:test-application]). We created an interface to the `ChronicleEngine`, the centerpiece of the service, and extended it to a replicated one, serving as a facade and the API for clients. We illustrated this in figure \ref{fig:chronicle-before-after}. Since we wrapped the relevant classes and interfaces of ChronicleDB into own extensions and facades, no changes in the core implementation of ChronicleDB where needed. To allow for distributed queries, the original query interface of the `lambda-engine` should be adjusted, since it exposes a cursor interface that is hard to wrap in a serialized `gRPC` command message. Instead of wrapping, it should be re-engineered for the distributed case. We limit our work to support replicated writes, so we did not touch this yet.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/chronicle-before-after.pdf}
  \caption[Re-engineered ChronicleDB on Raft]{Re-engineered ChronicleDB on Raft. The replication layer sits between the event store instances and the lambda engine.}
  \label{fig:chronicle-before-after}
\end{figure}

Note that the event data model in ChronicleDB is slightly different to our definition in section [@sec:consistency-choice]: In ChronicleDB, an event contains two timestamps $t_1$ and $t_2$, denoting the start and end of the _temporal validity_ of the event. This makes discussions about the event order somewhat more complex, but we won't go into it.

The full picture of the architecture is shown in figure \ref{fig:ha-chronicle-architecture}. The different parts of this architecture will be explained in the following sections.

\begin{figure}[H]
  \centering
  \includegraphics[width=1.5\textwidth, angle=90]{images/ha-chronicle-architecture.pdf}
  \caption{Architecture of a ChronicleDB cluster for high availability}
  \label{fig:ha-chronicle-architecture}
\end{figure}

#### Cluster Management {#sec:cluster-management}

In the center of the replicated ChronicleDB implementation sits the cluster management. The cluster management is crucial for managing nodes, partitions and instances of state machines in the cluster. 

<!-- describe what it does -->

The cluster manager acts as both a book keeper and as a service for provisioning nodes to new or initial partitions of a state machine instance. It

- Keeps references to all nodes, Raft groups, divisions, and state machine instances,
- Stores all metadata of the cluster, including information on the machines hardware, OS and load,
- Performs regular health checks using heartbeats (including failure detection),
- Serves Raft nodes with health info about the cluster,
- Provisions new partitions, such as for newly instantiated event streams, with load balancing,
- Acts as a gateway and routes requests to the Raft group of the according state machine and partition,
- Updates the _bootstrap lists_ of the clients (the list of known nodes of a cluster a client can message to),
- Acts as a service for clients to query the cluster state.

It therefore plays a crucial part in the architecture; without it, no write or read can happen at any node.

<!-- describe how it does it (running as a seperate Raft group) -->

The cluster manager is implemented as a replicated state machine itself, as it must provide at least the same level of fault-tolerance and availability as the actual application logic due to its crucial role in the architecture. The cluster management and actual application logic is divided into two instances of Raft servers, as shown in figure \ref{fig:management-server}. This guarantees that the management server is always up, even in case of faults of the application servers, which are detected by the management server. While the management Raft group runs on all nodes, ensuring an additional level of fault-tolerance, the actual application groups run only on the nodes with partitions covering them. Moreover, the cluster manager runs another state machine that serves a simple in-memory key-value store for additional metadata that we haven't yet implemented with type safety. This key-value store allows for single-nested key-value pairs. We use this to store and distribute additional node metadata, such as remaining disk capacity, which is exposed to the user via an API and a UI and can be used for alerting or auto-scaling.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/management-server.pdf}
  \caption[Management and application server]{A single deployment contains both a management and application server. The management server is responsible for all book keeping of the cluster.}
  \label{fig:management-server}
\end{figure}

In Apache Ratis, every instance of a `RaftServer` runs on its own port. Therefore, we need to provide two ports per node, one for messaging and replication of the application logic, and one for the cluster management[^meta-nodes]. While this adds complexity for cluster operators, it ensures that both servers are decoupled. This is similar to what other vendors are doing (InfluxDB, Apache IoTDB, see subchapter [@sec:previous-work]). Although one could imagine running cluster management and application logic on separate machines, we run them on the same machines for efficiency and complexity reasons. This is illustrated in figure \ref{fig:multi-raft-load-balancing}. In addition, there is one public port for client requests. In our example setup on AWS, as well as in our local Docker setup, we made both ports for cluster management and application replication private. We recommend this to reduce the risk of someone messing up the protocol messaging, and thereby the consistency of the system.

[^meta-nodes]: The literature commonly refers to a node partaking in cluster management as a _meta node_, since it manages, stores, and replicates the metadata of the system, in contrast to the _data nodes_, which replicate the actual payload data. A physical node can have both the role of a meta and data node at the same time.

The Raft log of the cluster management state machine does not persist every command, simply to avoid additional I/O and log compaction complexity for very frequent commands such as heartbeats.

##### Raft Groups, Clients and Servers.

Ratis provides three classes that are mandatory to run a cluster on Raft.

A _Raft group_ is a logical instance, containing a set of nodes that replicate a single state machine instance. Each physical node can participate in multiple Raft groups, meaning it can take on different roles at the same time. This relation is called a _division_. A division is a relational tuple describing a node, a Raft group, and the role of this node in this group.

A _Raft client_ is a client instance that can send requests to a single Raft group. On instantiation, the Raft client is given the Raft group ID and a bootstrap list of nodes to reach out to. Once a Raft client reached out to a single node, it receives information about the whole group and what the current leader is, and communicates directly to the leader from this point. A Raft client is mandatory for sending any message to a Raft group. A write request sent from a Raft client is written and replicated to Raft logs of a quorum of the nodes in the group, before it is applied to the state machine. Raft clients can be run on any `Java` service on any machine as long as they can reach one Raft server of a Raft group and know the ID of the Raft group to connect to.

A _Raft server_ is the instantiating of Raft on a single node. Ratis supports Multi-Raft, so a Raft server can run multiple Raft groups.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/ratis-classes.pdf}
  \caption[Multi-Raft in Apache Ratis]{Multi-Raft in Apache Ratis, here with a replication factor of 3}
  \label{fig:ratis-classes}
\end{figure}

##### Commands.

As we describe in subsection [@sec:messaging], all communication between nodes is done via `gRPC`. All possible commands and message schemas are defined in `Protocol Buffer` schema descriptions. This provides a clear, transparent and reliable interface for all communication with the cluster management server. The server supports the following commands:

- `REGISTER_PARTITION(string statemachine_classname, string partition_name, int replication_factor)` to register a new partition for a set of nodes and a state machine with the given replication factor,
- `DETACH_PARTITION(string partition_name)` which detaches a partition without deleting its persisted contents,
- `LIST_PARTITIONS(string statemachine_classname)` to list all registered partitions, either for all or a specific state machine,
- `HEARTBEAT(RaftPeerProto peer)` for a node (_peer_) to broadcast a heartbeat to let the cluster know of its presence.

In addition, the metadata state machine provides the following commands:

- `SET(string scope_id, string key, string value)` writes a value to a key for a certain scope. If the key is already defined, it is overwritten. The scope allows to define multiple instances of the key-value store, identified by the scope ID.
- `DELETE(string scope_id, string key)` removes the key from the scope, if existing,
- `GET(string scope_id, string key)` returns the value for the key of the given scope,
- `GET_ALL_FOR_SCOPE(string scope_id)` returns the whole map for a scope,
- `GET_ALL()` returns all maps for all scopes. 

<!--
##### Failure Detection.

\todo{Describe the heartbeat mechanism} 
-->

#### Event Store State Machine {#sec:event-store-state-machine}

The ChronicleDB event store is split in multiple event store instances, where each instance manages a particular event stream for a given event schema. In a standalone ChronicleDB deployment, these event store instances are managed by the `lambda-engine`, which maintains a map of references to the respective instances. In the basic approach, the list of available stream is managed with a metadata file on disk, next to the event streams. Each event stream itself is stored a set of files, which contain the block segments, the index, and secondary indexes, if any.

In the distributed approach of this work, the event store instances are managed by the cluster manager, which keeps book of all the registered event store instances as partitions. Each ChronicleDB event store instance is managed by a dedicated state machine instance. In this subsection, we examine our implementation and additional abstractions that we have built on top of Apache Ratis to improve maintainability and generalizability.

##### Formal Definition.

In its core, the state machine for an event stream can be formally described as follows (based on subsection [@sec:state-machine-replication]), under the condition that the only valid write operation is the insertion of one or multiple events:

$C$ is the set of all possible events $e$.

$S = \{\dots\}$ denotes all possible sequences $E$ of events.

Writing one event can be denoted as

$\delta(e, \langle \rangle) = \langle e \rangle$ where $\langle \rangle$ is the empty stream.

For non-empty streams:

$$\delta(e, \langle e_1, \dots, e_n \rangle) = 
  \begin{cases}
    \langle e_1, \dots, e_n, e \rangle & \text{if } e \geq e_n, \\
    \langle \delta(e, \langle e_1, \dots, e_{n-1} \rangle), e_n \rangle & \text{else }\text{ (out-of-order)}.
  \end{cases}$$

and writing events in batches can be described by

$$\delta((e'_1, \dots, e'_m), \langle e_1, \dots, e_n \rangle) = \delta (e'_m, \dots \delta(e'_1, \langle e_1, \dots, e_n \rangle )\dots)$$

##### Generic State Machine Instantiation.

Our architecture allows us to run not only instances of event stores, but instances for any arbitrary implementation of a state machine. This allows us to break down the replicated logic into multiple state machines for different service domains, which improves the performance of a complex system since this approach allows us to break coordination and linearization of operations where we don't need it. However, we must be careful not to decouple logic with a great amount of strong causal connections. We can either deploy the state machines from the same code base or seperately in a service-oriented approach. Both are valid approaches since the communication between different state machine instances is abstracted using Raft clients and Raft groups. They handle to build a connection with the appropriate group leader via `gRPC`, while the messaging between groups is reliable due to the downward and backward compatibility of message schema evolution in `protobuf`. We illustrated this in figure \ref{fig:decouple-state-machine-monolith}. It is also imaginable to introduce a message bus, i.e. using Kafka (or ChronicleDB itself) to decouple communication between state machine instances.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/decouple-state-machine-monolith.pdf}
  \caption[Decoupling a monolithic state machine]{Decoupling multiple service domains from a monolithic state machine that must not neccessarily be linearized into multiple state machine instances can improve the performance of the overall system}
  \label{fig:decouple-state-machine-monolith}
\end{figure}

Thanks to Multi-Raft in Ratis, each Raft group can run a different state machine implementation. A fresh state machine instance is registered as a single partition via the cluster management. You can instantiate such a instance using the `StateMachineProvider` interface. It bundles the state machine to be deployed, a descriptor for that instance/partition, the number of replicas to partake in that partition, and other domain-specific information such as the event schema. When issuing the creation of a new event stream for a given schema, the API creates such a provider and registers it to the cluster management, which then executes the provisioning by selecting the replicas with the lowest load, spawning a new Raft group for the requested state machine instance and writing it to the cluster managers log.

There is one state machine instance for each event store instance, which covers a single event stream. Currently, there is exactly one partition per state machine instance, as we don't support sharding yet. Our implementation of the `lambda-engine` makes use of the cluster manager to retrieve the set of instantiated event store state machines, so it can forward client requests for writes and queries to the respective event stream. 

##### The State Machine Interface.

Our state machine interface abstracts the original Ratis interface in multiple layers to improve the maintainability of the code and the system and to allow to implement new state machines with less effort and highest possible modularity.

In Ratis, the state machine API entices you to build a monolithic state machine, which quickly becomes hard to read and to maintain. With our approach, we enforce modularity and atomicity. Each abstraction layer of the state machine has a specific set of responsibilities (the layers are illustrated in figure \ref{fig:state-machine-interface} as well as in the UML diagram in figure \ref{fig:chronicle-uml-classes-annotated}):

- The abstract `ExecutableMessageStateMachine` class can be extended to provide lightweight state machines that uses `ExecutableMessages` together with `OperationExecutors` to decouple state and operations on the state. It simply references the `State` that is to be managed, the `StateManager`to manage the state, and the class of allowed `ExecutableMessages` on this state.
- The `StateManager` interface must be implemented to keep a reference on the `State`. Direct manipulations on the `State` object are prohibited, instead all allowed operations must be implemented via the `StateManager`. The `StateManager` is passed to `OperationExecutors`, exposing its interface for atomic state updates.
- The `State` interface should be implemented to describe how the state should be instantiated (what the empty state looks like), how it is persisted (references to the actual storage, such as the node's disk) and how snapshots are taken (for log compaction).

In Ratis, a state machine has at least two methods `applyTransaction` and `query`. 

- `applyTransaction` receives the `TransactionContext`, which includes the Raft log entry. The log entry itself includes the message payload for the execution on the state. Developers are responsible to update the last applied term-index themselves in case the operation was successful, otherwhise to rollback the state change (in case it is not atomic).
- `query` is similar to `applyTransaction` with the difference that it does not receive a log entry, but only the message payload. Query operations are not allowed to mutate the state, therefore, no entry is written to the Raft log.

We generalized the `applyTransaction` method. Developers do not longer need to update the term-index themselves. Instead, all operations must be written atomically asa pair of an `ExecutableMessage` together with an `OperationExecutor`. For the case of a failed execution, every `OperationExecutor` allows to provide a `cancel` method for neccessary rollbacks or compensations, in case an operation can not be designed atomically and/or idempotent.

For side effects (not on the state—this is prohibited—but rather for logging or notifications to the cluster manager) we added a `beforeApplyTransaction` respectively `afterApplyTransaction` method.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.75\textwidth]{images/state-machine-interface.pdf}
  \caption[Splitting up the state machine interface]{Splitting up the state machine interface into the state machine, the state manager, the state itself and message-executor pairs.}
  \label{fig:state-machine-interface}
\end{figure}

##### The Event Store Facade.

The actual event store instances that manage the event streams are prevented from direct access. They can only be accessed through the replication layer, which ensures consistency. For convenience, an event store facade hides the replication layer and offers the same API to producers and consumers as the original event store classes, since it implements the same interface. This follows the philosophy of strong consistency to render a replicated system to clients as it where a single-server system. We extended this approach to the `lambda-engine`, which offers an API to access multiple streams. However, we haven't managed to implement neither continuous queries nor batch processing yet (cf. subsection [@sec:implementation-limitations]).

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/event-store-facade.pdf}
  \caption[Facade hiding the actual event store]{The actual event store instances are hidden behind a facade. The replication layer wraps the event store instances, preventing direct access to them.}
  \label{fig:event-store-facade}
\end{figure}

##### Architecture and Message Flow.

We illustrate the class hierarchy and the message flow in the annotated UML diagram in figure \ref{fig:chronicle-uml-classes-annotated}.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/chronicle-uml-classes-annotated.pdf}
  \caption[UML class diagram for the replicated ChronicleDB event store]{UML class diagram for the replicated ChronicleDB event store state machine. The annotations show the message flow for an event insert operation. A non-annotated version can be found in the appendix.}
  \label{fig:chronicle-uml-classes-annotated}
\end{figure}

An insert operation includes the following components in several steps:

\begin{enumerate}[label=(\arabic*)]
  \item The \texttt{insert} of an event is requested via the HTTP API.
  \item The reference to the target stream (partition) is retrieved from the cluster management. It is necessary to use it to instantiate the Raft client so that it sends the request to the leader of the coresponding Raft group.
  \item The insert operation of the \texttt{BufferedReplicatedEventStore}is called, which is a facade for the Raft client. It implements the same interface as the actual event store and additionally provides a buffered insert operation.
  \item Once the buffer is flushed (meanwhile, other insertions could have been requested), a \texttt{InsertBulkEventsOperationMessage} is instantiated. Its payload contains the events to be inserted from the buffer. Before flushing, the events are ordered by event time. The message is then serialized using \texttt{Protocol Buffers} and sent via \texttt{gRPC} to the leader of the Raft group that manages the streams partition.
  \item The leader receives the message...
  \item ...and writes it to the Raft log. The log entry is now replicated to a quorum.
  \item Once the message is replicated to a quorum, the \texttt{EventStoreStateMachine} instances at each quorum node (and successive other nodes) apply the message, executing the \texttt{insert} command.
  \item The \texttt{insert} operation is finally executed on the event stream by the actual event store instance on all participating and available nodes.
\end{enumerate}

Note that in future work, the order of this operations may differ if the Raft log is optimized; i.e. a message is applied immediately on the state machine while writing a pointer reference only to the Raft log (cf. section [@sec:log-less]).

##### Commands.

The prototype implementation of the event store state machine supports the following commands on a stream:

- `PUSH_EVENTS(repeated<Event> events, bool is_ordered)` inserts the given list of events, which is optimized by telling ChronicleDB in advance if these events need to be reordered before inserting,
- `AGGREGATE(Range range, AggregateScope aggregate_scope, AggregateType aggregate_type, string attribute)` derives an aggregate of the given type (currently supporting `COUNT`, `SUM`, `MIN` and `MAX`), either on a attribute or global scope, for the given time range,
- `GET_KEY_RANGE()` returns the time range for the stream,
- `CLEAR()` returns the whole stream.

A query message is not yet implemented due to the complexity of the query interface and more challenges that we pointed out in subsections [@sec:system-design-limitations] and [@sec:implementation-limitations].

#### Log Implementation

The Raft log in Apache Ratis has all the properties that a Raft log must have: it is replayable, compactable and comes with a term-index pair for each log entry to mark the current leader term and the index of this entry for linearizability. It also denotes the high-water mark of the log with the `lastAppliedTermIndex`. In addition, Apache Ratis allows to implement the Raft log yourself, and independently from the state machine, by providing an interface for managing log data outside the Raft log. For data-intense applications, we recommend to do so: they provide a basic, non-optimized Raft log implementation that dumps the log entries to segmented binary files on disk, and reads them again from disk when feeding them to the state machines. This basic implementation is not efficient, slowing down data-intensive applications. In our evaluation, writing every event to the Raft log slows down the system tremendously, due to increased I/O and especially because the default log in Ratis requires a log entry to be written before it is sent to other nodes, effectively blocking the whole system on every single operation (this could be mitigated by parallelizing log writes and replication). As a key insight, the Raft log implementation is crucial for the performance of the whole system. 

 Some of the applications we investigated use efficient embedded storage engines such as _RocksDB_ that are optimized for high-throughput[^cockroach-rocksdb], some others wrap the log in a cache[^hashicorp-log-cache], and some even come with additional WAL logs, providing a multi-layer architecture of the Raft log. Even if this comes with some issues, we can learn from it to implement a better log. We use buffering to improve the performance of the system, avoiding too many writes to the log while at the same time sacrificing atomicity of the log entries. This approach is presented in subsection [@sec:buffered-inserts]. In the next paragraph, we also present an approach to the log that we haven't managed to implement in our prototype yet.

[^cockroach-rocksdb]: CockroachDB uses RocksDB for their log: https://github.com/cockroachdb/cockroach/issues/38322

[^hashicorp-log-cache]: Hashicorp Raft wraps a cache with a in-memory ring buffer around the Raft log to avoid I/O on recently written log entries, as the state machine, replication engine and log disk writer can immediately pick up the log entry from the buffer after an insert: https://github.com/hashicorp/raft/blob/main/log_cache.go 

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/chronicle-raft-log.pdf}
  \caption[Relation of the Raft log and the event store]{Relation of the Raft log and the event store. a) Appending events without allowing for out-of-order events always results in the same sequential order of the events in the Raft log (wrapped in operations) and the events in the event store, even if insertion and event time differ. b) With buffered inserts and without out-of-order insertions, the order correlation still applies, as the buffer contents are ordered before insertion and the buffer can only be served from a single leader node. c) As soon as out-of-order insertions are allowed, the order of the two logs given by insertion and event time is no longer guaranteed to correlate. This can break previously derived non-monotonic aggregates, as shown in figure \ref{fig:ooo-consistency}.}
  \label{fig:chronicle-raft-log}
\end{figure}

##### Log-Less Rafting. {#sec:log-less}

With the Raft log, we write the same event twice: Once to the Raft log, and once to the event stream (when it is replicated and committed). The messages end up being stored redundantly, and the duplicate writes use up additional I/O capacity, which slows down the system. It also violates the _"The log is the database"_ philosophy of ChronicleDB. So the question is: why can't the event stream just serve the log instead of creating a redundant Raft log?

- The Raft log contains not only events (actually insert commands), but also Raft commands (such as leader changes, network reconfigurations etc.) and in the future additional operations on the event stream (for example for schema evolution, but here we would recommend to just create a new stream).
- Not all entries in the log are already committed and applied to the state machine, similar to a WAL.
- With out-of-order events, the ordering of the Raft log and event stream can differ (see the previous discussions in section [@sec:consistency-choice]).

As we wrote, other applications use embedded storage engines for high-throughput to write the log. ChronicleDB is such a high-throughput embedded storage engine. Consequently, this leads to the consideration of using the ChronicleDB event store itself as the log. We can address this by reducing the Raft log to pointers, at least for the write operations. We use the event stream as the truth for the log and keep only pointers in the Raft log (next to the command, in this case `insert`). Once a message arrives at a node, the state machine immediately writes it into the event stream and at the same time a pointer to the Raft log. When reading the event stream, the system must read the high-water mark of the log to know which event stream entries are already committed, and is only allowed to return these. In case of uncommitted entries dropped from the log (e.g., after a network partitioning or due to some faults), the event stream entries after the high watermark are allowed to be overwritten. Overwriting entries is a new concept to ChronicleDB that must also be discussed from an index and storage layout perspective, therefore we haven't implemented this throughout this work. This approach must also take care of out-of-order events: it is not sufficient to read the high-water mark and allow to return all past entries in the event stream that have an event timestamp before the watermark, because this could return out-of-order entries that were not yet fully replicated to a quorum and committed. It is only allowed to return entries with an insertion timestamp before the entry with the high-water mark, which requires further thoughtful engineering to be efficient. In addition, no materialized aggregates are allowed to be returned that are based on uncommitted out-of-order entries. This is illustrated in figure \ref{fig:logless-raft}.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/logless-raft.pdf}
  \caption[Almost logless Raft]{Sketch of an almost logless Raft. a) Regular Raft log, with events only written to the stream once committed, but then, they are written twice. b) Almost logless Raft, where the Raft log only contains pointers to the actual events in the stream. Events are immediately written to the stream, but only those up to the high-water mark are allowed to be read by clients. c) This simple approach won't work if out-of-order events come into play.}
  \label{fig:logless-raft}
\end{figure}

Ignoring out-of-order events, snapshotting the state and compacting the log is also easy, because it will mean no more than just removing old pointers that we are not interested in and keeping the high-water mark as the pointer to the most recent stream event.

There are also more approaches to logless state machine replication in the literature, that goes way beyond this. So far, they are only suited to smaller key-value stores, as they manage a full sequence of states, but research here could be deepened for larger structures such as an event store [@skrzypzcak2020towards].

#### Buffered Inserts {#sec:buffered-inserts}

In his dissertion on Raft [@ongaro2014consensus], Ongaro describes batching and pipelining support for Raft log entries and explains their importance to system performance. Collecting and executing multiple requests (Raft log entries) in batches reduces I/O and network round-trips at a high level. Unfortunately, Ratis does not support batching itself. Therefore, we implemented a buffer ourselves to provide batching. In this subsection, we carry out implementation details, while we refer to subsection [@sec:buffer-theory] for more details. 

One caveat of our implementation is, that log entries lose atomicity: instead of having one log entry per event, a log entry resembles a batch insertion command. This is shown in figure \ref{fig:chronicle-raft-log} b). A better implementation would be to implement batching right in the log instead of outside the log. But, it is not that bad at all: many of the vendors we inspected suggest to producers to write events in batches anyway[^influx-batches]. With buffering inserts, we take the burden from the event producer to send in batches by enforcing that events are written in batches anyway. While this does not reduce network latency between the client and the ChronicleDB cluster, it reduces the inter-cluster latency by a significant order.

[^influx-batches]: InfluxDB, for instance, suggests as a best practice to write events in batches https://docs.influxdata.com/influxdb/v2.3/write-data/best-practices/optimize-writes/#batch-writes

##### Buffer Implementation.

Figure \ref{fig:ha-chronicle-architecture} illustrates the buffer architecture:

\begin{enumerate}[label=(\arabic*)]
  \item A client emits events to be inserted into the store. This happens either through a remote API (in this case, a \texttt{REST} API) or directly on one of the nodes via the `Java` API in case of an embedded design.
  \item The partition manager of the ChronicleDB engine finds the right partition for the insertion request and the current leader for this replica group to serve the request. The partition and leader mapping is cached for a performanant in-memory lookup. The request is sent to this leader, while the client is notified for direct access of the leader for subsequent requests.
  \item The inserts are added to a buffer that allows for synchronized concurrent inserts and writes. The buffer is implemented using a \texttt{LinkedBlockingQueue}. The buffer is flushed using the \texttt{drainTo} method of the \texttt{LinkedBlockingQueue} once it overflows or at least after a certain timeout after the last insert. The buffer is optional and both its maximum size and timeout are configurable.
  \item The flushed buffer content is inserted into the event store in a batch operation. Before inserting, the events are ordered in-place in the buffer to prevent out-of-order inserts in the given set. The flush and insertion happen in a single, atomic operation.
  \item The insert of the events happens in a batch. The insert operation, including the event payloads, is appended to the Raft log, replicated and applied to the state machine. Following the Raft consensus protocol, the insertion is acknowledged once a majority of the nodes committed the operation. The state machine manages a state object, that itself encapsulates the actual ChronicleDB event store index.
\end{enumerate}
 
\todo{Algo of the buffer}

##### Advantages of the Buffer.

- By ordering all events in a batch and ordering the batches itself (through the Raft log), we ensure linearizability.
- Our buffer implementation helps to increase the real-time consistency and to cope with out-of-order entries. The maximum size and timeout of the buffer can be configured in a way that it covers the inconsistency window, sorting out-of-order events in advance before inserting them into the actual event streams.
- The buffer also allows for concurrent writes by multiple clients, as multiple threads can write to the buffer, while its content is ordered on flush. This mitigates overlapping writes, as the buffer content is always ordered by event time, which is even more strict than the strong consistency guarantees that do not enforce ordering on concurrent writes.
- Writing events in a fire-and-forget mode to the buffer allows for very high throughput. With horizontal scaling, it is possible to keep a constantly high throughput rate (events/s). Thanks to the buffer, intermediate higher event emitting rates can be compensated, if the mean emitting rate stays below the maximum tolerated throughput. Otherwhise, the overall system slows down and probably crashes, as the stream may not converge (cf. subsection [@sec:time-bound-partial-consistency]).

##### Buffer Size Considerations.

There are multiple factors that influence the optimal buffer size:

- **Throughput**: Since we currently rely on the basic Raft log implementation of Ratis, a sufficient buffer size should be selected to allow for high throughput. In our evaluation, we demonstrate the impact of the buffer size on the overall throughput.
- **The inconsistency window**: The buffer size and timeout can be chosen to cover the inconsistency window, so there is a high chance to provide consistency to real-time applications even with out-of-order entries.
- **The RPO**: As mentioned in subsection [@sec:geo-replication], some organizations have a metric called the Recovery Point Objective (RPO), which specifies the maximum number of lost writes that the application can tolerate when recovering from a disaster. Since the buffer entries are transient and get lost on a crash, the buffer size must be chosen accordingly. Considering some _graceful shutdown_ techniques, the risk of data loss on a crash can be further reduced.

##### Limitations.

- As we pointed out in subsection [@sec:buffer-theory], we should acknowledging each write to the producer by keeping a callback reference for each write. This is important when a client needs the guarantee that some events are written before it continues sending further events.  Implementing this is challening and we haven't implemented that in the prototype. At present, only a successful write to the buffer is confirmed, but this does not guarantee a successful write to the quorum.
- Our buffer implementation always orders events by event time, more precisely: by the start timestamp. If a different sequence is required, it should be made configurable accordingly.
- To avoid writing big batches of events that are out-of-order when multiple clients write to the cluster, only the current leader of a Raft group should be allowed to accept these writes and add them to its buffer. For our evaluation, we ignored the multi-client case, therefore we are missing this restriction in our prototype yet. Currently, the buffer sits in the event store facade, so each node is allowed to write to its buffer and file a batch request to the leader node once the buffers are flushed. We strongly recommend to enforce the single leader semantics.

#### Messaging between Raft Nodes {#sec:messaging}

Apache Ratis supports a few messaging protocols for communication between nodes (for messages written to the Raft logs as well as Raft-internal messages, such as for leader election). We have chosen `gRPC`, since it comes with some benefits:

- It runs on `HTTP/2`, which has a lower latency compared to `HTTP/1.1`, and is more secure thanks to `TLS 1.2`.
- It uses `Protocol Buffers` (also referred to as `protobuf` for short) as its _interface description language_, which allows us for efficient, schema-safe messaging with out-of-the-box serialization. 

All Ratis messages are defined as `protobuf` schemas, too, which makes it transparent and easy to understand what nodes exchange during Raft protocol instances.

Commands issued against state machines are represented via messages, and we define the schema of these messages with `protobuf`. Every such command must be functional, serializable and side-effect free. The messages can be sent by `RaftClients`, as shown in subsection [@sec:event-store-state-machine]. Example messages have been shown in the same subsection.

Messages are categorized by write and read messages. In Ratis, writes are called transactional messages, while reads are called queries, as illustrated in figure \ref{fig:state-machine-interface}. This may be misleading since it does not provide transactions in the sense of ACID databases. It is rather to be understood as a single, atomic command execution: it is either executed and acknowledged on the state machine or no state transition happens in case of a failure. Yet this depends on the implementation of the commands and executors, and it is certainly possible to design message executors that are not atomic or deterministic, or that do not roll back changes in case of failure.

##### Messaging and Execution Architecture.

One goal of our messaging architecture is to avoid writing monolithic state machine code. We want to decouple the replication protocol and the application logic in case we need to switch to a replication protocol other than Apache Ratis one day. Our architecture of the messages and the execution interface shown in figure \ref{fig:chronicle-uml-classes-messages} is replication-protocol agnostic. This allows us to reuse the same messaging interface for future improvements and even completely different consistency layers, e.g., the executors could be used in a job worker like fashion to parallelize workload in eventual consistent operations, such as queries that can be served at the same time. In addition, we have schema-safety thanks to `protobuf` code generation.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/chronicle-uml-classes-messages.pdf}
  \caption[UML class diagram for the messaging architecture]{UML class diagram showing the classes and interfaces used to describe messages and operations. The message interface is implemented to wrap the schema classes generated from protobuf interface definitions, while the executor interface is implemented to describe the actual execution logic. The messages and execution logic itself is decoupled from the state machine code. This seperation allows to describe the operations agnostic from the replication protocol and implementation.}
  \label{fig:chronicle-uml-classes-messages}
\end{figure}

##### Serialization.

As we already layed down, every message must be serializable so it can be sent between Raft nodes and persisted in the Raft log. It also allows us to sent these from clients that are not written in `Java`. Since our main command is the `insert` command, we must also serialize the events that form the payload of this message. Since the payload of an event is variable, depending on the event schema, serializing it with `protobuf` is not trivial. Instead of defining a variable `protobuf` schema to serialize events, we simply use the built-in event serializer of ChronicleDB. This also prevents additional deserialization steps, since we can just pass the binary representation of the event through the message layer, until it is applied by the state machine.

##### Adding New Messages.

Adding new messages (i.e., new supported commands for state machines) requires the following steps:

\begin{enumerate}[label=(\arabic*)]
  \item Definition of the message schema in \texttt{protobuf}.
  \item Generating \texttt{Java} code (message classes) out of the schemas.
  \item Implementing the \texttt{ExecutableMessage} interface in case of an all-new state machine implementation, for example \texttt{EventStoreOperationMessage}. This wraps the generated message instances from \texttt{protobuf} and adds additional metadata and type hierarchies that are not available in \texttt{protobuf}.
  \item Expanding the factory methods of the parent message object to create the various message instances (for example, to insert an event).
  \item Implement the \texttt{OperationExecutor} interface, one for each \texttt{protobuf} message type defined. \texttt{OperationExecutors} must implement the \texttt{apply} method which defines how this single operation message is to be executed on the state machine. This method receives the payload of the \texttt{protobuf} message and a reference to the state of the state machine. There are two additional interfaces, where exactly one must be implemented: either the \texttt{TransactionOperationExecutor} or the \texttt{QueryOperationExecutor}. Only the first is allowed to mutate the state of the state machine, and it should be an atomic operation. To handle failed, non-atomic operation executions, the \texttt{TransactionOperationExecutor} interface provides a \texttt{cancel} method to implement neccessary rollbacks or compensations, in case that an operation can not be designed to be atomic and/or idempotent. It returns a \texttt{Future} containing the response of the operation for the client or an error message if it fails.
\end{enumerate}

##### Future Improvements.

In future work, we would wrap this RaftClient in a `Java` SDK as it gives us `gRPC` messaging for free and we don't need to reimplement all commands again for `REST`/`HTTP`. `REST` runs on `HTTP/1.1` which adds additional network round-trip and (de-)serialization time, thus slowing down the overall system. `gRPC` is more efficient than REST as it leverages `HTTP/2` and works well with our already existing binary `protobuf` definitions, without introducing additional (de-)serialization steps. However, this requires us to re-engineer the gateway implementation of the cluster manager; the clients must first ask the cluster manager which node they should talk to before sending the request. We sketched this in figure \ref{fig:chronicle-java-sdk}. Currently in our demo application, this is handled on the server side by the `ChronicleEngine` for simplicity reasons. This also allows us to avoid another additional serialization step when deploying ChronicleDB as a service, since clients could directly serialize into ChronicleDB's target format instead of a `JSON` string first, which must be deserialized and serialized again on server side.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.5\textwidth]{images/chronicle-java-sdk.pdf}
  \caption[Java SDK and gRPC API for ChronicleDB]{The bottleneck of json serialization of events and additional network round-trip can be reduced by introducing a gRPC API. This requires a discovery step where the client SDK asks the cluster for the leader and Raft group ID of the stream it wants to talk to.}
  \label{fig:chronicle-java-sdk}
\end{figure}

There is great potential for the application of model-based programming and code generation. We have shown how to add additional messages, which can quickly become cumbersome. With the `protobuf` definitions and additional `Java` annotations, one could imagine to generate a huge portion of this code—including messages, message wrappers, factories, and operation executors.

We also recommend readers to keep an eye on the further development of `gRPC` regarding support for the new `HTTP/3` protocol[^grpc-http3], which is built on top of `QUIC` and `UDP` instead of `TCP`. `QUIC` was approved as an IETF standard in 2021 [@iyengar2021quic], while `HTTP/3` was approved recently in June 2022 [@iyengar2022http3]. `HTTP/3` can further reduce latency, especially in settings with a lot of roundtrips, such as in Raft.  

[^grpc-http3]: The request for `HTTP3` support in `gRPC` is tracked in this issue on GitHub: https://github.com/grpc/grpc/issues/19126

#### Partitioning and Horizontal Scalability {#sec:multi-raft-partitioning}

The throughput of a single event stream instance has an upper bound, based on I/O capabilities of the underlying machine, the OS, concurrent access to the ressources of that machine, the implementation of the ChronicleDB index and lastly—usually with the most impact—the replication protocol and the underlying network. To be able to scale beyond this limit and to overcome the latency trade-offs introduced by Raft, we need to introduce partitioning (cf. section [@sec:partitioning]).

With partitioning, we can make the distributed ChronicleDB horizontally scalable. For our prototype, we implemented a basic approach that runs one partition per event stream, with simple load balancing based on the number of available nodes and required replicas.

##### Multi-Raft.

Ratis allows for Multi-Raft, which supports running multiple Raft groups on a single clusters. When you run multiple basic Raft instances, one for each partition (stream), on a single cluster, each of these comes with their own leaders and heartbeats. Unless we constrain the Raft group participants or the number of topics, this creates an explosion of network traffic between nodes. In Multi-Raft, this is handled differently. There are dedicated leaders for each group, but only a single heartbeat round-trip for all nodes. A node, once it partakes in at least one Raft group, becomes a Raft server. The Raft server is responsible for sending and/or receiving AppendEntries messages and hearbeats (empty such messages). The Raft server itself is logically split into multiple Raft divisions. One division represents one Raft server taking part in one Raft cluster. On the division level, each server can be either a leader, follower or candidate. We illustrate our basic Multi-Raft approach in figure \ref{fig:multi-raft-architecture}.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.8\textwidth]{images/multi-raft-architecture.pdf}
  \caption[Partitioning and replication with multiple Raft groups]{Partitioning and replication with multiple Raft groups. When a client accesses a a Raft group on the cluster for the first time, it accesses any available node it has on its bootstrap list. If this node is not the leader, it updates the clients bootstrap list. Subsequently, the client writes and queries this leader node until a new leader has been voted. A node can participate in multiple Raft groups, having either a leader, follower or candidate role for this group.}
  \label{fig:multi-raft-architecture}
\end{figure}

##### Load-Balancing.

With Multi-Raft, we can achieve very basic load-balancing by evenly distributing replicas on available nodes. We achieve this using a `PriorityBlockingQueue` that orders nodes by the number of already registered groups on that nodes. We adopted this approach from the Apache Ratis examples. This is illustrated in figure \ref{fig:multi-raft-load-balancing-basic}, where 6 partitions (including metadata/cluster management) are distributed on 5 nodes with a replication factor of 3 (note that the cluster management is deployed on each node to increase its availability, fault-tolerance and locality).

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/multi-raft-load-balancing-basic.pdf}
  \caption[Basic load-balancing with Multi-Raft]{Basic load-balancing with Multi-Raft. Partitions with a replication factor of 3 are evenly distributed across 5 nodes. Metadata is replicated on every node to ensure cross-cluster disaster resilience and fault tolerance and allows the partitions on the respective nodes to directly read the metadata on its own node.}
  \label{fig:multi-raft-load-balancing-basic}
\end{figure}

In the next paragraphs, we take a look at further possible improvements, that did not make it into our prototype.

##### Time Splits.

Since event streams are append-only structures, historical data quickly becomes immutable (at least after the inconsistency window). In addition, ChronicleDB splits the stream storage by time splits (see section [@sec:time-splitting] for reference). We can take further advantage of this by introducing partitioning by those time splits. The time splits in ChronicleDB sit in their own $\textrm{TAB}^+$ index trees, as well as in seperate files, which makes it naturally suitable for partitioning.

Time splits allow for improved read scaling. Historic time splits are read-only (we will assume that out-of-order events do not happen in historic splits), thus the single-leader requirement of Raft can be dropped here. Furthermore, historic time splits could also be replicated using a different replication protocol, such as primary-copy, or simply by installing Raft snapshots, since we don't care about write consistency anymore. Generally, they would naturally "grow", once the index tree is "chopped" when a new time split should be created. When this happens, a new partition could be instantiated immediately (or better in advance to stay available during this period) and the cluster manager routes writes to the new partition, while the old partition is locked for writes and made available to be read on all nodes. We sketch such a configuration in figure \ref{fig:multi-raft-load-balancing}. Moreover, the replication factor can also be increased for historical splits by installing snapshots, which further reduces read latency and provides data locality in the case of multiple availability regions.

We haven't implemented partitioning by time splits yet, since we need some strategies to execute queries and aggregate requests across time splits.

Partitioning by time splits is also available in other time series databases and event stores, such as IoTDB (there, it is called _time slices_) [@wang2020iotdb].

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/multi-raft-load-balancing.pdf}
  \caption[Load-balanced partitioning using time splits]{Partitioning using time splits with a replication factor of 3, load balanced on 5 nodes. For historic time splits, the single-leader requirement could be dropped since they are read-only, resulting in that every replica can serve query requests. The superscript here describes if a node is either a leader or follower, while the subscript denotes the starting timestamp of the split.}
  \label{fig:multi-raft-load-balancing}
\end{figure}

##### Sharding (Write Splits).

While partitioning by time splits lowers the read latency, sharding (effectively resulting in _write splits_) lowers the write latency, allowing the event store to scale with the number of writing clients. We illustrate a possible setup for sharding in figure \ref{fig:multi-raft-historic-splits-sharding}.

Sharding in consistently ordered append-only structures is not trivial. Since sharding would only lower write latency when writes across shards are no longer linearized, write consistency must be sacrificed. Instead of strong consistency, each partition would only offer sequential consistency at maximum. Therefore, we haven't implemented this in our prototype. While sacrificing linearization on writes, we need to re-introduce linearization on reads, at least for the recent time split. 

- When historic time splits are generated, the shards of the current split could be merged in the background into a buffer (i.e., a ring buffer) while another thread writes the immediate results into the historic split.
- To achieve linearizability on recent splits when querying shards, we need to merge shards of the same event schema in the query engine. Thanks to the sequential ordering of each shard, we can just use in-memory merging and cache our results for already merged intervals (the cache must be partially invalidated if out-of-order events occur). For instance, merging using a min-heap can be done with $\mathcal{O}(N k \log k)$ time and $\mathcal{O}(Nk)$ space complexity, where $N$ is the number of events per shard (in case of balanced shards) and $k$ the number of shards. With tumbling time windows and parallelization, we can greatly improve the time complexity, since individual time windows be merged independently. We illustrate this in figure \ref{fig:event-stream-merging}.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-historic-splits-sharding.pdf}
  \caption[Multi-raft cluster with historic time splits and sharding]{A multi-raft cluster with historic time splits and sharding of the current time split to improve write throughput and latency}
  \label{fig:multi-raft-historic-splits-sharding}
\end{figure}

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/event-stream-merging.pdf}
  \caption[Two event stream shards merged on a query]{Illustration of two event stream shards merged on a query with tumbling windows}
  \label{fig:event-stream-merging}
\end{figure}

##### Consistency Across Partitions.

With our partitioning approach, we do not provide strong consistency across streams, since we do not ensure that the order of events to be written into each event stream matches their insertion time (cf. subsection [@sec:system-consistency-across-partitions]). This is fine, since we are only interested in the actual event time. If there is causality between streams, this only becomes relevant when querying across multiple such streams and should be handled by the distributed query processor, as shown in figure \ref{fig:query-consistency} (also see the previous paragraphs for a parallelizable merge approach). We haven't implemented this in our prototype in the context of this work.

For the sharding case, it looks as follows: even though sharding (in combination with horizontal scalability) significantly increases overall throughput, it causes a stream to lose write consistency as we forgo inter-shard coordination—which would only increase latency again. We need to restore the consistency again when querying the shards, i.e. by merging. How this could look like architecture-wise is illustrated in figure \ref{fig:chronicle-consumer-producer}. We also described this in the sense of causal consistency in subsection [@sec:stream-causal-consistency].

Even across streams for different event schemas, consistency cannot be guaranteed because eeach stream is handled by its own partition and thus its own Raft group. We limit our implementation to the one-partition-per-stream case, thus we are not able to evaluate the impact of sharding on the overall througput.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.9\textwidth]{images/chronicle-consumer-producer.pdf}
  \caption[Relation between producers and consumers in ChronicleDB]{Schematic illustration of the relation between producers and consumers, and how consistency across shards may look like. The merge step is also illustrated in figure \ref{fig:event-stream-merging}.}
  \label{fig:chronicle-consumer-producer}
\end{figure}

##### Rebalancing.

Once a node crashes, it should be replaced by another available node, to ensure a quorum can still be maintained for each Raft group. There are two cases of node crashes from the perspective of a single Raft group: The crash of a follower and the crash of the leader. On both, the network reconfiguration protocol of Raft must come into play to allow a new node to join the group, while the protocol differs on both cases. The network reconfiguration in Raft allows for rebalancing without compromising availability. We illustrate these cases in figures \ref{fig:multi-raft-follower-fails}} and \ref{fig:multi-raft-leader-fails}. Note that our prototype is missing the rebalancing yet.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-follower-fails.pdf}
  \caption[Rebalancing in the event of a follower failure]{Rebalancing in the event of a node failure that leads to a raft group with too few replicas, while the leader stays intact. a) A fault makes a node crash (fail-stop). The meta quorum detects this crash and notifies the leader of this group. b) The leader requests a membership change from the meta quorum. The quorum selects a node based on balancing rules and triggers a network reconfiguration. The node is added as a new follower to the group. The leader then installs the current state snapshot on this new follower replica.}
  \label{fig:multi-raft-follower-fails}
\end{figure}

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-leader-fails.pdf}
  \caption[Rebalancing in the event of a leader failure]{Rebalancing in the event of a leader failure. a) As the followers no longer receive heartbeats from the leader of their group, they start a vote. The winner of the vote becomes the new leader of this group. b) The cluster manager can now assign a new follower similar to the case in figure \ref{fig:multi-raft-follower-fails}.}
  \label{fig:multi-raft-leader-fails}
\end{figure}

<!--
#### Fail-Over Design

TODO Network reconfiguration, together with chronicleDB failover
-->

#### Edge-Cloud Design

By adding a replication and partioning layer to ChronicleDB, while keeping its capabilities of running standalone on a single server or embedded in another application, ChronicleDB meets the basic requirements to run in an edge-cloud architecture. 

We illustrated the edge architecture, including its three layers, in section [@sec:edge-computing]. ChronicleDB fits into all three layers, thanks to its different deployment modes. An example setup for the edge-cloud is illustrated in figure \ref{fig:chronicle-edge-architecture}.

##### Terminal Layer.

ChronicleDB is originally designed and optimized to run on the terminal layer. It can be used as a "centralized storage system for cheap disks running as an embedded storage solution within a system (e.g., a self-driving car)" [@seidemann2019chronicledb]. It then runs as a lightweight library tightly integrated in the application code, with zero network latency, allowing for the highest throughput on the perspective of a single stream.

##### Edge Layer.

On the edge layer, ChronicleDB allows to aggregate a multitude of events from devices in the terminal layer. In general, the complexity of the events between the layers increases, while the volume per stream decreases: While the terminal layer produces raw events, the edge layer provides filtered and aggegrated events to the cloud layer that are ready to be consumed in reporting applications. The basic idea is to run _Event Processing Agents_ (EPA) in the sense of Complex Event Processing on the edge layer and use ChronicleDB in standalone mode as an event sink. By running ChronicleDB on the edge, it provides highest data locality and event processing power close to clients, providing lower response times since dedicated ressources on the edge can be used for processing that are not available in the cloud to these extents.

##### Cloud Layer.

On the cloud layer, ChronicleDB is deployed in the distributed version that we presented in this work. In the cloud, the goal is to scale with the number of devices, respectively streams, from the terminal layer, while providing high-availability and fault-tolerance. While ChronicleDB in the cloud does not provide the same high throughput on a per-stream basis as embedded on a device in the terminal layer, it shows its advantages in horizontal scaling, as it beats the throughput of the embedded version on a multi-stream basis.

##### Between the Layers.

To close the gap between the ChronicleDB instances on the different layers, one could either use an event processing solution, a pub/sub system or continously querying one ChronicleDB deployment with another. The latter is only suitable between the edge and the cloud, and to stream edge or cloud events into the terminal layer, since we can not query streams from devices on the terminal layer in a pull fashion—this is better to be implemented in a push fashion. We refer at this point to the related literature.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/chronicle-edge-architecture.pdf}
  \caption[Architecture of ChronicleDB in an edge-cloud setting]{Potential architecture of running ChronicleDB in an edge-cloud setting, next to and in combination with stream processing systems. It can act both to ingest and serve data before processing, as well as a data lake after processing. Different applications can query the ChronicleDB event store, either ad-hoc or continously. Both real-time and non-real-time systems can read streams from ChronicleDB, which can result in different consistency levels required.}
  \label{fig:chronicle-edge-architecture}
\end{figure}

### Evaluation Application {#sec:test-application}

To test and evaluate the implementation of the replicated event store and the middlewares supporting it, a test application is needed. In the context of this work, a synthetic application is implemented to perform trade-off studies. The application is build with the following stack:

- Spring Boot as the `Java` Framework to provide ChronicleDB on a Raft as a service.
- React to provide a lightweight user interface frontend for evaluation purposes.
- Maven and Webpack to build the whole application.
- Docker to run it in containers.

This setup allows us to evaluate ChronicleDB both locally and in the cloud.

#### Supported ChronicleDB Feature Scope

We do not support all baseline features of ChronicleDB now in our distributed prototype. We support the following two main operations that we used to evaluate our implementation:

- **Insertions**: We support inserting events, either as single events, buffered, or in custom defined batches.
- **Aggregation**: We support basic ad-hoc aggregation by implementing the `AggregatedEventStore` interface and using the aggregate feature of the `TABPlusTree` under the hood. We use this to evaluate the read performance.

##### Limitations.

We do not support queries at the moment, neither ad-hoc nor continuous queries. See section [@sec:implementation-limitations] for details. 

<!-- We also do not support the creation of new streams on runtime, since we haven't managed to complete implementations here. Streams must be defined via code, and are created on server startup. -->

#### Running Inserts

The evaluation application does not use the `HTTP REST` API to insert events, as `REST` runs on `HTTP/1.1` which adds additional network round-trip and (de-)serialization time, thus slowing down the overall system (cf. subsection [@sec:messaging]). Instead, we run an evaluation service on one node of the cluster, that continuously generates random events and inserts them into the replicated event store to simulate a continuously emitting sensor device. This evaluation service can be started and stopped via a dedicated API, given params to control the velocity of inserts. In addition, we have written some simple `python` functions to call this evaluation service from a `Jupyter` notebook, not matter if ChronicleDB runs on a local or remote cluster. The notebook allows us and others to run the evaluation reliably and repeatable. This notebook can be found in the appendix.


#### Running a Cluster

Deploying and running a ChronicleDB cluster looks similar to how you run any other distributed database cluster. When starting a node, you specify its ID in the cluster, its storage directories, the ports for metadata/management, replication and the public API, and tell each node the adresses of other nodes in the cluster (the bootstrap list of at least 3 nodes to form a quorum) so they can find each other (while this list is further populated by the cluster manager once a quorum of nodes can talk to each other).

For instance, a single node on a local cluster (single machine) is started this way:

```
PEERS=n1:localhost:6000,n2:localhost:6001,n3:localhost:6002

ID=n1
SERVER_PORT=8080
META_PORT=6000

java -jar target/chronicledb-raft-0.0.1-alpha.jar \
--node-id=$ID \
--server.address=localhost \
--server.port=$SERVER_PORT \
--metadata-port=$META_PORT \
--storage=$STORAGE_DIR/$ID \
--peers=$PEERS
```

This makes it easy to run a ChronicleDB cluster in a container environment, such as docker. A whole cluster can be deployed with a single docker-compose.

<!--

Deploying a node locally to run in a test cluster is done as follows:

```
PEERS=n1:localhost:6000,n2:localhost:6001,n3:localhost:6002

ID=n1
SERVER_PORT=8080
META_PORT=6000

java -jar target/chronicledb-raft-0.0.1-SNAPSHOT.jar \
--node-id=$ID \
--server.address=localhost \
--server.port=$SERVER_PORT \
--metadata-port=$META_PORT \
--storage=$STORAGE_DIR/$ID \
--peers=n1:localhost:6000,n2:localhost:6001,n3:localhost:6002
```
TODO explain

Respectively on all nodes

#### Running in Docker Containers

- TODO simple cluster start with docker-compose

```
services:
  n1:
    build:
      context: ./
      dockerfile: Dockerfile
    image: chronicledb-raft:latest
    command: --node-id=n1 --server.address=n1 --storage=/chronicledb-raft --peers=n1:9999,n2:9999,n3:9999
    ports:
     - "8080:8000"
  n2:
    build:
      context: ./
      dockerfile: Dockerfile
    image: chronicledb-raft:latest
    command: --node-id=n2 --server.address=n2 --storage=/chronicledb-raft --peers=n1:9999,n2:9999,n3:9999
    ports:
      - "8081:8000"
  n3:
    build:
      context: ./
      dockerfile: Dockerfile
    image: chronicledb-raft:latest
    command: --node-id=n3 --server.address=n3 --storage=/chronicledb-raft --peers=n1:9999,n2:9999,n3:9999
    ports:
      - "8082:8000"
```

TODO feeding in env file for properties
-->

#### User Interface

The prototype comes with a simple user interface that helps to access various functions the cluster manager, such as checking the vitality of the cluster. It serves an overview over all running raft groups and the current roles of each node in that groups. It also allows to manage event streams and to run aggregations on them, mainly to continuously observe the throughput in evaluation settings. It is also used to evaluate and demonstrate the aggregations themselves.

It allows navigation between cluster nodes and can also be used to check the impact of the cluster's strong consistency from the user perspective, simply by running the user interfaces of all nodes in multiple browser windows and observing how changes are rendered simultaneously.

\todo{Screenshot of a Raft Group}

\begin{figure}[H]
  \centering
  \includegraphics[width=0.95\textwidth]{images/chronicledb-screenshot-1.png}
  \caption[Screenshot of the ChronicleDB event store UI]{Screenshot of the ChronicleDB event store UI}
  \label{fig:chronicledb-screenshot-1}
\end{figure}

### Implementation Summary

In this subchapter, we presented our implementation of a replication and partitioning layer for ChronicleDB. We went over the system design, starting with a thorough discussion of consistency models and their impact on ChronicleDB as well as on clients. We worked out the requirements for the replication layer and decided on a consistency model that would meet those requirements. We decided for strong consistency, since this allows us to guarantee safety to clients, at least in the absence of out-of-order events. We discussed the impact of out-of-order events on write and read consistency, as well as various mitigation strategies, and decided to implement a write buffer as a simple and lightweight solution. We laid out the trade-offs of this solution and made suggestions for possible future improvements. Finally, we explained our choice for a replication protocol and a library that implements this protocol in `Java`. Based on this library, we presented the technical architecture of our prototype implementation, which also includes an evaluation service.

In the next subchapter, we will use this evaluation service to evaluate our implementation, using the original standalone ChronicleDB implementation as a benchmark.

<!--
TODOS:

- Write readme.md
- Write unit + integration tests
-->

\pagebreak

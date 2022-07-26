## System Design and Architecture {#sec:system-design}

<!--
Developed architecture / system design / implementation: 1/3
• start with a theoretical approach
• describe the developed system/algorithm/method from a high-level point of view
• go ahead in presenting your developments in more detail
-->

- This section is about the solution design and implementation
- First list high-level requirements (strong consistency, availability, fault-tolerance...) just reference the previous chapters
    - TODO describe that I opted for strong consistency here [@gotsman2016cause]    
    - TODO describe my reasoning for this
    - Reference Calm
    - if possible (and trade-off ok), strong consistency is always prefered
    - Many other db vendors provide strong consistency (anecdotal proven that it works, TODO reference previous work)
        - Especially with raft or similar protocols
        - Even in high throughput scenarios
        - Multiple strategies to solve throughput issues like multi-raft + partiioning, less replicas for the write partition (time split) but increase replica count for read-only splits (easy as those only few OOO inserts into old splits does not affect whole performance when looking at the big picture)
- Then detailed requirements (how the user interacts with it, requirements to the API, infra and tech requirements)

- TODO show table of the different dependability levels that are minimum for this

- TODO show why not a CRDT or using optimistic replication (even if we could achieve optimistic repl with "conflict" resolution via OOO (there are no real conflicts due to no random writes, so mention this "no-conflict other then OOO case" here)); show why strong consistency; why raft
- explain scalability issues and why they aren't that bad and how to tackle them

- Then show design: What platform, what libraries (reference next ratis section)...

\todo{list Algorithms (heartbeat, load balancer, partitioning...) like in https://software.imdea.org/~gotsman/papers/unistore-atc21.pdf}

TODO does all of this still makes sense??? Merge with my other notes! Event store is NOT monotonically growing as defined in CALM - the order of events matter, depending on the application using the store!!

TODO discuss here, and also at the end when we evaluate our impl of executableMessages:
- append-only computing by Pat Helland 2015: We have immutable aggregates: Time series data + domain events. TODO read that paper
ACID - Associative, Commutative, Idempotent, Distributed (CALM):
- Validate our operations if they are
- Possible to apply eventual consistency?
- insertEvent() could apply: it inserts this one specific event with the given timestamp. But there is no mechanism in place that identifies unique events, there we could add the same event twice at the same timestamp
- raft log is idempotent ?: every operation (e.g. insertEvents) is unique due to term:index, and order of execution does not matter (is commutative) for inserts and also for createAggregates, but it matters when we consider deletes (and queries?). Therefore, we should have at least causal consistency (?)
- But even if inserts alone are communitative, OOO is too expensive. 
- In case the event store would only allow for inserting events, it would be monotonic
- So it depends on which set of operations we allow 
- We could model delete operations differently, so they could also be monotonic (= everything is an insert), by just logging them. The tree would then consist of both addEvent and deleteEvent items, and just by observing its final state we know which events are actually there. (It seems like this may does not make sense in our use cases)
- Also see https://www.slideshare.net/SusanneBraun2/keeping-calm-konsistenz-in-verteilten-systemen-leichtgemacht
- Also describe the trivial, derived monotonic aggregates thing
- TODO reference these in the conclusion. We won't build an eventual consistent prototype. But it could make sense. It can be built without any coordination mechanism between nodes if we agree for them to only use monotonic operations.
- the ohlc of a stock in a certain timeframe is a trivial derived aggregate that is suitable for our example
- insertEvent is monotonic. Aggregates are also trivial, derived monotonic aggregates -> can use eventual consistency
- but addStream, cluster management, metadata etc. must be strong consistent! (like InfluxDB did it) -> this is not monotonic (see CALM)

TODO describe that createSecondaryIndex and creation of "materialized user-defined aggregates" is also an operation in the raft log, but we won't focus on it yet - "ChronicleDB computes these aggregates incrementally as new events arrive" - so we don't care

## Challenges

\epigraph{Problems are not stop signs, they are guidelines.}{--- \textup{Robert H. Schuller}}} 

TODO brief enumeration of all major challenges

- Strong consistent ordering
- Or at least causal
- OOO
- PACELC: Consistency vs. Latency
- It violates "The log is the database" by introducing a new log (raft log)
    - For future work: Transient raft log; after succesful commit, entries in log are gone; index is always latest snapshot. Also marry raft log and OOO write-ahead log. If we need to derive the log again: with OOO, not possible
- Multi-Threaded Event Queue Workers and Partitioning
    - We don't know how the decoupling affects the possible performance that can be achieved with multiple streams through the ChronicleDB own event queue and worker architecture, as with partitioning, the relation is no longer one-queue-per-stream, but one-partition-per-stream

Throughout the next sections, we will show how we tackled these challenges, and in chapter [@sec:conclusion], we will show how these could be further mitigated in the future.

## Library Decision Considerations

- Building it from ground up makes sense if you want full control and adjust the protocol to perfectly fit your use case (TODO find and cite the paper/tool that mentioned that, it must have been mongo or couchbase)
- Adjusting the protocols allows you to tickle out the last ounce of performance, but oftentimes under the cost of losing the formal verification (TODO reference to raft verification mention in 03b)
- This thesis is about a proof that it works, so we use something existant to leverage implementation power of the OOS community and save time to build the proof
- We use Java for simplicity, as ChronicleDB is written in Java. For best performance and future-proof support (at the time of this writing), it is recommended to implement the library yourself or to use one of the popular Golang implementations: etcd/raft or hashicorp/raft. One could even use the Gorum framework in Golang (with some adjustments as in [@pedersen2018analysis])
- It is also possible to build upon a library / an API and also leveraging the library in a way that allows you to maximize throughput
- TODO List other libraries in short

## Consistency Model and Replication Protocol Choice

- Refer to [@sec:consistency-decisions]
- Refer to append-only and CALM [@sec:calm]

- TODO describe why strong consistency and finally raft
- TODO reference criteria from 04a
- TODO reference CAP and PACELC and describe our trade-offs

TODO "why consistency is the better choice for most databases"
https://www.cockroachlabs.com/blog/limits-of-the-cap-theorem/

TODO our buffer introduction reduces latency and increases availability; but reduces the consistency from linearizable to sequential (after a write, it can not be immediately read. As the buffer content is not readable, only after a flush, but then on all nodes, it is not eventual consistent, but sequential.)

The actual consistency model to decide for depends on the use case of the distributed database system and especially in this case for the distributed event store.

We assume to "built for a set of use cases where data safety is a top priority"

"strong consistency not designed to be used for every problem. Their intended use is for topologies where queues exist for a long time and are critical to certain aspects of system operation, therefore fault tolerance and data safety is more important than, say, lowest possible latency and advanced queue features."

"to avoid OOO, which is expensive, linearizability is a must; otherwhise, the number of random I/Os increases drastically and consequently the ingestion performance suffers, resulting in back pressure to the producer at least and data loss due to dropped events at worst (see [@sec:chronicle-out-of-order])"

TODO paragraph "Most common use cases for event stores"

CEP and ESP

I think event sourcing also requires strong consistency (event is readable after write and no suprising events appearing in the past/no OOO)

TODO does complex event processing (CEP) always require strong consistency? Would otherwhise a CEP agent never now when to run a aggregation over events? I don't think so. It does only rely on patterns it recognizes, then it is a monotonic problem - but only if patterns never rely to detect events that are NOT there in between of others, and when consistency is achieved at least in the sliding window timeframe. The patterns often work with wildcards to ignore arbitrary events in between, but I think it depends on the patterns and use cases: For example, if I recognize a pattern of a temperature drop but in fact it is a temp raise but I haven't yet received the high temp event in between  

TODO compare with what
- https://fauna.com/blog/why-strong-consistency-with-event-driven
- and https://drops.dagstuhl.de/opus/volltexte/2019/10307/pdf/LIPIcs-ICDT-2019-5.pdf
says

"A first important remark is that event streams are noisy, and one does not expect the events matching a formula to be contiguous in the stream. Then, a CEP engine needs to be able to dismiss irrelevant events."

"However, we must not assume that the data will always arrive in the correct time sequence. When we perform the search for the complex event, it may happen that the user data arrives earlier than the GPS data. " - NO strict consistency across streams

"For monotonic problems, new information will not change that fact. Additional facts can only result in additional derived events being
detected: the output (computed events) grows monotonically with the input (factual events)."

"EP deals with events that occur several times redundantly, concurrently and unreliably chained. Moreover, the logic of occurrence is not sharply but fuzzily defined."

--> So for CEP, lower consistency levels are ok: CEP patterns ARE a monotonic problem (reference CALM here)

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/complex-event-processing.pdf}
  \caption{In complex event processing, events from multiple streams are fuzzily correlated to derive monotonic aggregates}
  \label{fig:complex-event-processing}
\end{figure}

"Traditional event streaming applications (_Event Stream Processing_, ESP) deal with a single stream of data arriving in the correct time order. An example would be algorithmic trading, where a simple ESP application could analyze a stream of pricing data and decide whether to buy or sell a stock. ESP applications do not normally include event causality or event hierarchies. "

--> For ESP, it is not! Actually, for ESP, OOO must be prohibited, as OOO violates consistency. (The user looks at the data at one time t1 and than at t2 where t2 > t1 and suddenly a set of past events has grown; while a "sell" or "buy" signal has been raised that assumed wrongly because it relied on events NOT beeing there)

TODO is this always true for CEP? Couldn't someome build a CEP rule that relies on the same non-existence property?

For ESP use cases like algorithmic trading, we have causally related events in a single stream. What was already observed must not change: if new information appears in the past, it will change existing derived facts (sell or buy signals) and render them wrong.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/ooo-consistency.pdf}
  \caption{For non-monotonic derived events or aggregates, out-of-order events must be prohibited, as they could render the derived events wrong: what was already observed must not change. a) A non-monotonic derived aggregate is created based on previous facts (events). b) When observing the event stream later again, an out-of-order event occured which invalidated the previous derived aggregate and created a new aggregate.}
  \label{fig:ooo-consistency}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.1\textwidth]{images/ohlc-ooo-problem.pdf}
  \caption{Algorithmic trading as an example for strong consistency requirements. a) A candlestick pattern was detected in the trading events in a order book, causing a buy alert to be sent (and executed). b) At a later time, the past candlesticks are displayed differently due to some out-of-order events, so that the buy alert is now wrong - but it is already too late. This reduces trust in this hypothetical trading platform and its reputation.}
  \label{fig:ohlc-ooo-problem}
\end{figure}


Event store as order book ledger, as seen at times t1 and t2

Event sourcing and complex event processing require at least causal consistency and, depending on the use case, causality stretches across all events of a certain event source or stream, requiring at least sequential consistency... (? TODO is that true?)

Aggregations of event properties for idempotent and commutative operations (such as a sum or a mean) do not require strong consistency / can have weaker consistency...

"Data loss due to system failures or system overload is generally not acceptable"

In event processing literature, the consistency levels are also heavily dependant on the use case and requirements of the final end user:
https://arxiv.org/pdf/cs/0612115.pdf

With our partitioning approach, we don't provide linearizability across streams, so we don't ensure the order of events to be committed to the respective event store instances / indexes to be according to their insertion time. This is ok as we only care about the actual event time.

### Consistency From the Client’s Point of View

- OOO: Assumes non-linearized client-server communication
- Happens if we allow for fire-and-forget (async comm between client and server)
- Buffer can reduce this drastically
- What if multiple clients (= devices) write to the same schema?
-> Easy solution: We could use one stream per client/device
- If we go for a single stream: We are linearized regarding ingestion time (but not for overlapping requests, which is the characteristic here)
- But not for event time (no strict consistency), so we can expect a lot of OOO
- But: The buffer also helps here to avoid OOO. Strict concistency is too expensive: Assuming the clocks of all devices are the same (which is rare; we only have that with infra in full control (cf. Google example)), we would still need to order events before storing in the store, but we won't know if there are still events to come (due to network latency of some clients). This would require extremely expensive coordination between clients. This is only of theoretical interest: we don't need that in practice due to OOO handling and the buffer

"However, we must not assume that the data will always arrive in the correct time sequence. When we perform the search for the complex event, it may happen that the user data arrives earlier than the GPS data. "

#### Transaction Handling

- TODO no transactions so far, which is good for us. No need to think about linearizing transactions and making them atomic.
- Only atomic inserts
- If we need transactions, especially across partitions, refer to [@sec:partitioning] and [@sec:coordination-free-replication]

""Currently, there is a lack of systems supporting the above-described workload scenario (write-intensive, ad hoc temporal queries, and fault-tolerance). Standard database systems are not designed for supporting such write-intensive workloads. Their separation of data and transaction logs generally incurs high overheads."

No transactions, but a log -> But still a very less overhead then with real transaction logs (with locking mechanisms)

## Raft Implementations

> TODO just the list from the raft github

## Apache Ratis

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

## Replication in ChronicleDB

- This section describes the actual implementation of replication in ChronicleDB with Apache Ratis.

- TODO better title for this chapter about the concrete implementation?

### Requirements

- List of all requirements/use cases
- TODO table showing consistency decision, dependability requirements...

### Overall Architecture & Design

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

#### Event Store State Machine

- Standalone Event Store wrapped
- Replicated Event Store as a facade / to be used as the client to the cluster
- TODO think of moving it out of spring boot, use as standalone lib
- TODO if reasonable, list algos

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
  \caption{TODO}
  \label{fig:chronicle-raft-log}
\end{figure}

#### Buffered Inserts

TODO compare raft log and event index
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
  \caption{Architecture of a ChronicleDB cluster for high availability.}
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

### Partitioning using Multi-Raft Groups

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

\paragraph{Time splits.}

In figure xxx, $A_{t_1}^L$ denotes a... when we restrict OOO for historic time splits, we could also replicate them leader-less, as there won't be any writes (TODO aggregate indexes on older splits)... and read from every available node... 

Time splits allow for read scaling; but for chronicleDB we need some strategies to execute queries and aggregates across time splits. (TODO describe strategies; TODO image showing query range |---| with one full and one half split; full split will just moved into memory (= the root of the index tree) while for the half split, the right leaf of the tree needs to be found)


TODO describe the benefit in read scaling with time splits (while writes on the same stream won't scale, as all of them (if we restrict OOO on historic splits) happen on the most recent split) with numbers if possible

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-load-balancing.pdf}
  \caption{Partitioning using time splits with a replication factor of 3, load balanced on 5 nodes. Metadata is replicated on every node (at least on every node in the bootstrap list) to ensure cross-cluster disaster resilience and fault tolerance and allows the partitions on the respective nodes to directly read the metadata on its own node.}
  \label{fig:multi-raft-load-balancing}
\end{figure}

( TODO is this extra hard replication on metadata neccessary? -> may describe possible byzantine tolerance. But what if you scale with hundreds of nodes? -> discuss this in conclusion)

\paragraph{Sharding (Write splits).}

TODO also mention hash split keys to have write-scaling.
This allows to scale with the number of writing clients. 
Once a historic time split can be created (and a new set of write splits), the current split can be merged in the background by another job (if current latency allows for it) or just marked as historic splits.
The write splits allow for faster writes, but when querying or aggregating, right strategies to resolve the operation across the shards need to be found. Every shard only offers sequential consistent data; they need to be merged to return the actual linearized stream. But thanks to the sequential ordering of each shard, we can just use in-memory merging, e.g. with a min-heap in $\mathcal{O}(N k \log k)$ time and $\mathcal{O}(Nk)$ space complexity, where $N$ is the number of events per shard (in case of balanced shards) and $k$ the number of shards (TODO reference)... which benefits from smaller time splits and also tumbling windows) and intelligent caching (TODO is there a better merge strategy for reads? The problem is: We write while we read; but if we exclude OOO we can guarantee that a cached read range won't change! problems only occur with OOO) 

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-historic-splits-sharding.pdf}
  \caption{A multi-raft cluster with historic time splits and sharding of the current time split to improve write throughput and latency}
  \label{fig:multi-raft-historic-splits-sharding}
\end{figure}



\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/event-stream-merging.pdf}
  \caption{Illustration of two event stream shards merged on a query with tumbling windows}
  \label{fig:event-stream-merging}
\end{figure}

TODO illutrate query over historic trees

|--- leaf --- root --- root --- |

<!-- tumbling windows 

https://ordina-jworks.github.io/kafka/2018/10/23/kafka-stream-introduction.html

-->

<!-- See for discussions around shard capacity and max throughput https://pt.slideshare.net/frodriguezolivera/aws-kinesis-streams?next_slideshow=true -->


TODO glossary with new words in the margin

#### Rebalancing

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-follower-fails.pdf}
  \caption{Rebalancing in the event of a node failure that leads to a raft group with too few replicas, while the leader stays intact. a) A fault makes a node crash (fail-stop). The meta quorum detects this crash and notifies the leader of this group. b) The leader requests a membership change from the meta quorum. The quorum selects a node based on balancing rules and triggers a network reconfiguration. The node is added as a new follower to the group. The leader then installs the current state snapshot on this new follower replica.}
  \label{fig:multi-raft-follower-fails}
\end{figure}

TODO leader fails

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/multi-raft-leader-fails.pdf}
  \caption{Rebalancing in the event of a leader failure. a) As the followers no longer receive heartbeats from the leader of their group, they start a vote. The winner of the vote becomes the new leader of this group. b) The cluster manager can now assign a new follower similar to the case in figure \ref{fig:multi-raft-follower-fails}.}
  \label{fig:multi-raft-leader-fails}
\end{figure}

TODO reference raft voting mechanism here again

TODO rebalancing when node comes back

### Fail-Over Design

Lorem...

### Messaging between Raft Nodes using gRPC and Protocol Buffers

> TODO here or earlier?

- Describe briefly the messaging format
- ExecutableMessages with Executors framework
- 1:1 protobuf message per java message wrapper
- protobuf: the message. Java: How to apply the message's command on the state machine's state
- Makes it scalable, modular, message-based instead of monolithic state machines
    - If suitable depends on requirements. If state machine operations should be encapsuled and a closed set, just wrap a StateManager around it and reference it in the MessageExecutors
- High potential for model based programming / code generation (could generate executors and java message objects from proto+annotations)

### Edge-Cloud Design

TODO what is edge cloud? [@cao2020overview]

\todo{System design illustration of embedded dbs + edge + cloud cluster}

\todo{Rephrase; may break it in}
\paragraph{Terminal layer.} The terminal layer, also refered to as the _sensing layer_, consists of all types of devices connected to the edge network, including mobile terminals and many Internet of Things devices (such as sensors, smartphones, smart cars, cameras, etc.). In the terminal layer, the device is not only a data consumer, but also a data provider. 
\paragraph{Edge layer.} The edge layer supports the access of terminal devices downward, and stores and computes the data uploaded by terminal devices. Standalone time-series database on industrial PC (= original standalone ChronicleEngine) with EPAs...
\paragraph{Cloud Layer.} The cloud computing center can permanently store the reported data of the edge computing layer, and it can also complete the analysis tasks that the edge computing layer cannot handle and the processing tasks that integrate the global information. Distributed event store in raft cluster mode


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

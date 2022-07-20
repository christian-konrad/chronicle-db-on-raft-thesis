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

The actual consistency model to decide for depends on the use case of the distributed database system.

to avoid OOO, which is expensive, linearizability is a must


Event sourcing and complex event processing require at least causal consistency and, depending on the use case, causality stretches across all events of a certain event source or stream, requiring at least sequential consistency... (? TODO is that true?)

Aggregations of event properties for idempotent and commutative operations (such as a sum or a mean) do not require strong consistency / can have weaker consistency...

#### Transaction Handling

- TODO no transactions so far, which is good for us. No need to think about linearizing transactions and making them atomic.
- Only atomic inserts
- If we need transactions, especially across partitions, refer to [@sec:partitioning] and [@sec:coordination-free-replication]

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

#### Buffered Inserts

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
\paragraph{Terminal layer.} The terminal layer consists of all types of devices connected to the edge network, including mobile terminals and many Internet of Things devices (such as sensors, smartphones, smart cars, cameras, etc.). In the terminal layer, the device is not only a data consumer, but also a data provider. 
\paragraph{Edge layer.} The edge layer supports the access of terminal devices downward, and stores and computes the data uploaded by terminal devices. Standalone time-series database on industrial PC (= original standalone ChronicleEngine) with EPAs...
\paragraph{Cloud Layer.} The cloud computing center can permanently store the reported data of the edge computing layer, and it can also complete the analysis tasks that the edge computing layer cannot handle and the processing tasks that integrate the global information. Distributed event store in raft cluster mode


TODO draw edge-computing diagram for chronicleDB

\todo{Rephrase}
- This implementation is designed to fit three deployment models, one for each layer of the edge computing architecture [@cao2020overview]: 
1. Embedded time-series database with highly efficient file-based storage on edge appliance like Raspberry PI on the _terminal layer_ (= original embedded event store/index)
2. Standalone time-series database on industrial PC (= original standalone ChronicleEngine) with EPAs... for the _edge layer_
3. Distributed event store in raft cluster mode for the _cloud layer_

- It therefore has both a standalone, embedded version, i.e. to be deployed directly on the IoT devices to be lightweight and ultra-fast, and the replicated solution to be deployed in the cloud for high-availability, fault-tolerance etc etc that can scale with the number of devices / streams
- What is currenlty missing is the sync between the embedded and the cloud one (TODO is this ROWA or Primary-Copy?); a good approach would be to have a Event Processing Agent (EPA, TODO reference) in the sense of Complex Event Processing (CEP, [@buchmann2009complex]) in between to aggregate the raw events to useful ones (the output of the sensors ), maybe built with Apache Spark, Kafka, RabbitMQ...

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

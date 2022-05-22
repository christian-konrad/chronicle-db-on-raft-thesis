## System Design and Architecture {#sec:system-design}

<!--
Developed architecture / system design / implementation: 1/3
• start with a theoretical approach
• describe the developed system/algorithm/method from a high-level point of view
• go ahead in presenting your developments in more detail
-->

- This section is about the solution design and implementation
- First list high-level requirements (strong consistency, availability, fault-tolerance...) just reference the previous chapters
    - TODO describe that I opted for strong consistency here [gotsman2016cause]
    - in background or later in implementation?
    - TODO describe my reasoning for this
    - if possible (and tradeoff ok), strong consistency is always prefered
    - Many other db vendors provide strong consistency (anecdotal proven that it works, TODO reference previous work)
        - Especially with raft or similar protocols
        - Even in high throughput scenarios
        - Multiple strategies to solve throughput issues like multi-raft + partiioning, less replicas for the write partition (time split) but increase replica count for read-only splits (easy as those only few OOO inserts into old splits does not affect whole performance when looking at the big picture)
- Then detailed requirements (how the user interacts with it, requirements to the API, infra and tech requirements)

- Then show design: What platform, what libraries (reference next ratis section)...

\todo{list Algorithms (heartbeat, load balancer, partitioning...) like in https://software.imdea.org/~gotsman/papers/unistore-atc21.pdf}

## Library Decision Considerations

- Building it from ground up makes sense if you want full control and adjust the protocol to perfectly fit your use case (TODO find and cite the paper/tool that mentioned that, it must have been mongo or couchbase)
- Adjusting the protocols allows you to tickle out the last ounce of performance, but oftentimes under the cost of losing the formal verification (TODO reference to raft verification mention in 03b)
- This thesis is about a proof that it works, so we use something existant to leverage implementation power of the OOS community and save time to build the proof
- It is also possible to build upon a library / an API and also leveraging the library in a way that allows you to maximize throughput
- TODO List other libraries in short

## Consistency Model and Replication Protocol Choice

- TODO describe why strong consistency and finally raft
- TODO reference criteria from 04a

## Popular Implementations of Replication Protocols

> TODO list them here or in previous work?

- Reference to previous work chapter

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

### Overall Architecture & Design

<!-- Now, I present my results -->

- Stack:
    - Spring Boot
        - Why Spring Boot
    - Apache Ratis
    - Google Protocol Buffers
    - Standalone/Embedded ChronicleDB Event Store + ChronicleEngine

- pretty architecture diagrams

- Architecture in overview: Communication layers, client + server architecture, node architecture, how to run the system, ... afterwards the details in own sections

- Describe target package/library ecosystem (replicated event store as lib, consumed by spring)

#### Cluster Management

- Similar to LogCabin (see previous work)
- The management quorum
- Bookkeeping of available nodes and balancing partitions
- Heartbeats and health checks
- Registering of Partitions for StateMachines
- Additional RaftServer (with own port)
- Explain how to startup the cluster similar to https://github.com/logcabin/logcabin/blob/master/README.md

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

### Simplified API for Apache Ratis

> This should already be covered in previous sections (cluster management + state machine)

- StateMachineProviders
- PartitionInfo
- ClusterManager

### Partitioning using Multi-Raft Groups

- Basic vertical partitioning by stream, no sharding
- One raft group = one partition containing an instance of a certain state machine, distributed over n nodes (n = replication factor)
- PriorityBlockingQueue for simple load balancing
    - Naive implementation (balanced per absolute # of partitions, not by time splits or per actual load); better partitioning approaches see background#Partitioning and Sharding + conclusion
- Show diagram of partitioning approach


### Replicated Microservice using Apache Ratis and Spring Boot

> May omit this. This is about providing my interface as a library, which is nice-to-have, but not subject of this thesis

// TODO consider to update title; as Microservices in general should be stateless and therefore replication does not make sense.
It is more about replicating a state or storage that is consumed by services.

### Messaging between Raft Nodes using gRPC and Protocol Buffers

- Describe briefly the messaging format
- ExecutableMessages with Executors framework
- 1:1 protobuf message per java message wrapper
- protobuf: the message. Java: How to apply the message's command on the state machine's state
- Makes it scalable, modular, message-based instead of monolithic state machines
    - If suitable depends on requirements. If state machine operations should be encapsuled and a closed set, just wrap a StateManager around it and reference it in the MessageExecutors
- High potential for model based programming / code generation (could generate executors and java message objects from proto+annotations)

### Running in Docker Containers

- TODO simple cluster start with docker-compose

### Running on Kubernetes

- TODO only cover it in this section if I get a working prototype running
- Helm Charts

### End-User APIs

> This may be covered in previous sections

#### Java API

- Replicated Chronicle Engine
- Buffer Settings

#### HTTP/REST

### Management User Interface

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

# Design and Implementation {#sec:implementation}

- This section is about the solution design and implementation
- First list high-level requirements (strong consistency, availability, fault-tolerance...) just reference the previous chapters
    - TODO describe that I opted for strong consistency here [gotsman2016cause]
    - in background or later in implementation?
    - TODO describe my reasoning for this
- Then detailed requirements (how the user interacts with it, requirements to the API, infra and tech requirements)

- Then show design: What platform, what libraries (reference next ratis section)...

## Popular Implementations of Replication Protocols

> TODO list them here or in previous work?

- Reference to previous work chapter

## Apache Ratis

- Repo, maybe mvn link, version 2.1.0 [ratisGithub2022] 

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

### Overall Architecture

> TODO pretty architecture diagrams

#### Cluster Management

- The management quorum
- Bookkeeping of available nodes and balancing partitions
- Heartbeats and health checks
- Registering of Partitions for StateMachines
- Additional RaftServer (with own port)

#### Event Store State Machine

Lorem ipsum...

#### Log Implementation

- Commands
- Replayable
- Transactions vs Queries
- Acts kind-of write-ahead log, but buffered
- Snapshot: Pointer to last committed event in the store
- Persisting the log expensive
- TODO compare with other databases write-ahead logs
- OOO challenges

#### Buffered Inserts

- > TODO pretty architecture diagram
- TODO reasoning, performance and concurrency considerations
- TODO compare the 2 buffer approaches (blocking vs. non-blocking)
- TODO explain impact of buffer sizes
- TODO explain similarities with Raft Log Buffer

### Simplified API for Apache Ratis

- StateMachineProviders
- PartitionInfo
- ClusterManager

### Partitioning using Multi-Raft Groups

- PriorityBlockingQueue for simple load balancing
    - Naive implementation (balanced per absolute # of partitions, not by time splits or per actual load); better partitioning approaches see background#Partitioning and Sharding + conclusion

### Replicated Microservices using Apache Ratis and Spring Boot

// TODO consider to update title; as Microservices in general should be stateless and therefore replication does not make sense.
It is more about replicating a state or storage that is consumed by services.

### Messaging between Raft Nodes using gRPC and Protocol Buffers

Lorem ipsum...

### Running in Docker Containers

- TODO simple cluster start with docker-compose

### Running on Kubernetes

- TODO only cover it in this section if I get a working prototype running

### End-User APIs

#### Java API

- Replicated Chronicle Engine
- Buffer Settings

#### HTTP/REST

### Management User Interface

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

\pagebreak

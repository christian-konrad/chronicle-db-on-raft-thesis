## Previous Work {#sec:previous-work}

\epigraph{If I have seen further it is by standing on the shoulders of Giants.}{--- \textup{Isaac Newton}}

- ... as far as current research goes into theoretical details of what is possible with Raft, seeing it in action gives insights into practical usage and therefore current limits and possibilities...

### Popular Applications of Replication Protocols

#### LogCabin

Written by the author of Raft... [@ongaro2015logcabin]
... this thesis' implementation for metadata and cluster management follows similar principle... see Implementation Section 

#### Apache Kafka

##### Zookeeper

- Discontinued in KIP-500: Self-Managed Meta Quorum with Raft, see next section

##### KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum (Raft)

- TODO summarize [@kafka2022kip500]

- Kind-of Raft

#### CockroachDB

- Raft

#### Camunda Zeebe

- TODO add stuff from our pages
- https://github.com/camunda/zeebe/tree/main/raft/src/main/java/io/zeebe/raft/state
- docs... etc

#### RabbitMQ

- Raft in Quorum Queues [@rabbitmq2021quorum]

#### ElasticSearch

- Some-kind-of Raft

#### MongoDB

- Kind-of Raft

#### etcd / Kubernetes

- Raft

#### Apache Ozone

Apache Ozone is a... [@apache2022ozone]
The replication layer of Apache Ozone is built upon Apache Ratis, a library for..., []
Apaache Ratis is the library of choice for the implementation of a replicated event store presented in the next chapter.

#### Other Mentionable Implementations

> TODO should summarize, not an own section per tool

- Hadoop
- Neo4j
- Couchbase
- Apache Spark
- Apache Flink
- Apache Cassandra

### Popular Event Stores and Time-Series Databases

#### TimescaleDB

#### InfluxDB

#### Prometheus

#### Apache Kafka

#### Other Mentionable Implementations

#### Use Cases and Differentiation to ChronicleDB

ABC

\pagebreak

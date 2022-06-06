## Previous Implementations {#sec:previous-work}

\epigraph{If I have seen further it is by standing on the shoulders of Giants.}{--- \textup{Isaac Newton}}

- ... as far as current research goes into theoretical details of what is possible with Raft, seeing it in action gives insights into practical usage and therefore current limits and possibilities...

### Criteria for Review and Comparison

### Popular Applications of Replication Protocols

#### LogCabin

Written by the author of Raft... [@ongaro2015logcabin]
... this thesis' implementation for metadata and cluster management follows similar principle... see Implementation Section 

#### Apache Kafka

##### Zookeeper

- Is Primary-Copy Replication (TODO refer to 03a)
- External agent
- Provides a risk as a single point of failure
- Ends up in Primary-Secondary Replication
- Discontinued in KIP-500: Self-Managed Meta Quorum with Raft, see next section
- TODO provide the reasoning from Apache here why Zookeeper is considered bad vs. raft/self-organized quorum
- Zookeeper is used in some/many (?) other Apache Projects:
- https://mesos.apache.org/ 

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
- Strongly consistent (TODO refer to consistency types in 03a)
    - Consistent + Partition-Tolerant (CP)
- Pull-based consensus

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
    - Availability+Partition Tolerance
- Redis (https://github.com/RedisLabs/redisraft), still working on it, not production-ready
    - https://redis.com/blog/redisraft-new-strong-consistency-deployment-option/ 
    - Consistency+P

#### Comparison Results

Lorem ipsum...

### Popular Event Stores and Time-Series Databases

Lorem ipsum...

#### EventStoreDB

Consensus; is it raft?
https://developers.eventstore.com/server/v21.10/cluster.html#cluster-nodes

#### TimescaleDB

Lorem ipsum...

#### InfluxDB

https://s3.amazonaws.com/vallified/InfluxDBRaft.pdf
https://www.influxdata.com/blog/influxdb-clustering/

What is stored via consensus?
Only Metadata:

- Cluster membership
- Databases
- Retention Policies
- Users
- Continuous Queries
- Shard Metadata

What is not stored via consensus?

- The time-series data itself, the indexes and schemas


To not store the data has been introduced with a newer version, before that, all data went through raft (<= v0.9-rc20)
    - Potentially high performance
    - Global schema which meant improved user experience
    - Easier to reason about when all data in consensus
- In practice it was inefficient and complex to implement correctly.
- Approach was abandoned by 0.9.0

Planned design in 0.10.0: Distinct Raft subsystem + data-only nodes

https://www.influxdata.com/blog/multiple-data-center-replication-influxdb/

- primary-copy (?) for  Data Center Replication, Not built-in:
https://github.com/influxdata/influxdb-relay 
This project adds a basic high availability layer to InfluxDB. With the right architecture and disaster recovery processes, this achieves a highly available setup.

https://github.com/toni-moreno/syncflux

Lorem ipsum...

#### Apache IoTDB

[@wang2020iotdb]

- https://iotdb.apache.org/
- Raft (TODO only for meta or also for data?)
- Meta Group, Data Group
- https://github.com/apache/iotdb/issues/3954
- https://github.com/apache/iotdb/tree/master/consensus
    - It is heavily WIP, but it's raft consensus is built on top of Ratis, same as for our implementation
    - But they built a layer around it, to be agnostic of the protocol and framework https://github.com/apache/iotdb/pull/5939
        - https://github.com/apache/iotdb/blob/master/consensus/src/main/java/org/apache/iotdb/consensus/ConsensusFactory.java
        - One API for all protocols
    - They aim to use different protocols for different parts of the db (meta, region management, data...)

#### Prometheus

Lorem ipsum...

#### Apache Kafka

TODO PubSub etc., Differentiation to event stores

#### Other Mentionable Implementations

Lorem ipsum...

#### Use Cases and Differentiation to ChronicleDB

Lorem ipsum...

#### Comparison Results

Lorem ipsum...

\pagebreak

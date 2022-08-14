# Evaluation {#sec:evaluation}

\epigraph{\hfill I've always liked code that runs fast.}{--- \textup{Jeff Dean}}

<!--
Measurement results / analysis / discussion: 1/3
• whatever you have done, you must comment it, compare it to other systems,
evaluate it
• usually, adequate graphs help to show the benefits of your approach
• caution: each result/graph must be discussed! what’s the reason for this peak or why have you ovserved this effect
-->

In this work, an implementation of the Raft consensus protocol was proposed for replication of ChronicleDB, a high-throughput event store. It has been shown (TODO reference section) that Raft is strongly consistent, so a trade-off in performance is to be expected. In theory, the larger the number of replicas, the higher the data availability; however, at the same time, the cost of replication (which comes at the expense of performance) increases. This section addresses the evaluation of this implementation, specifically the cost of replication, by quantifying the costs and benefits that come with this approach. The challenge of the implementation is to find an optimal trade-off between the cost of replication and the availability of data.

## Evaluation Goals

"The main evaluation goals of this work were to test the additional cost of our techniques in terms of the amount of latency and reduced throughput introduced in the system..."


TODO measure all of the following dependability properties:

- Availability: readiness for correct service
- Reliability: continuity of correct service,
- Safety: absence of catastrophic consequences on the user(s) and the
environment,
- Confidentiality: absence of unauthorised disclosure of information,
- Integrity: absence of improper system state alterations,
- Maintainability: ability to undergo repairs and modifications.

TODO plot a reliability function! As of https://www.researchgate.net/figure/Comparison-of-experimentally-observed-and-modeled-RAFT-performance-with-clusters-of_fig4_321140059

\todo{Adapt question-style of evaluation from https://software.imdea.org/~gotsman/papers/unistore-atc21.pdf}

\todo{If time, construct simulation of edge-computing with standalone nodes pushing to replicated cluster}

<!-- 
TODO measure:
* Network partition case
* Performance boost on horizontal scale (= partitioning, feeding in data in multiple groups)
* Read performance
-->

<!-- TODO use evaluation style of http://www.diva-portal.org/smash/get/diva2:24228/FULLTEXT01.pdf -->

## General Performance Considerations

- TODO do we need this section or just refer to consistency section?
- Consistency models faced with different performance considerations

"horizontal scaling has the same issue as replication
for availability: both require maintaining replica consistency.
In general, maintaining strong consistency is achievable but
expensive"

TODO cite and compare with performance considerations of the raft dissertation https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf p 140 ff

TODO reference the throughput graph from raft chapter, phrase it as "expected throughput decline"

TODO also compare against throughput chart from zookeeper paper https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf and http://www.cs.cornell.edu/courses/cs6452/2012sp/papers/zab-ieee.pdf

To evaluate this, the implementation is benchmarked against... as described in the following section.

TODO also compare with cassandra
"ChronicleDB outperformed Cassandra by a factor of over 200
in terms of write performance on a single node. In other words: Cassandra would need at least 200 machines to compete with our solution."

TODO consider to evaluate sharding (maybe in future work)



## Setup for Comparison

TODO do it like in the koerber_diss.pdf  but also with the synthetic data

- Local Environment (Single Machine, Many Nodes (3, 4, 6))
- AWS, (Kubernetes, Docker?)
- Parameters to set for health checks (measure how long does it take for a regular node to start up)
- Test dataset (random data + data from DEBS Challenge)
    - Test series of 1, 100, 10k and 1m events
- Jupyter Notebook (publish on GitHub and reference here)

- TODO not just analyze throughput on a single stream. This gets slow indeed.
- also evaluate horizontal scalability! Which means multiple different streams (with partitioning).
    - Expected: Faster than standalone (mention here standalone could benefit from multithreaded/concurrency, but only to a certain limit (CPU cores, shared memory etc.)) 
    - TODO reference/citate literature on this

The whole experimental setup and evaluation can be found and repeated by yourself in the Jupyter notebook in the appendix.

## Throughput on Different Cluster Settings

"To evaluate our system, we benchmark throughput when the system is saturated"

TODO first describe expected latency decrease due to network round-trip/maximum wide area delay of replica nodes (do some math here)

"In Figure 5.2 we show a comparison between the version using replication and the one without replication. The latency cost of the version using replication can be clearly observed and is expected since..." https://www.diva-portal.org/smash/get/diva2:843227/FULLTEXT01.pdf

### Comparison of Cluster Sizes

TODO also aggregates

### Comparison of Buffer Types and Sizes

TODO without buffer

writing every event to the Raft log slows down the system tremendously, due to increased I/O and especially because the default log in Ratis requires a log entry to be written before it is sent to other nodes, effectively blocking the whole system on every single operation.

### Throughput on a Distributed Remote Cluster

// TODO AWS stuff

### Exceeding the Upper Throughput Threshold

We stress-tested the cluster by increasing the write load in a way that the mean throughput exceeds the maximum throughput for a longer period of time.

The stress-test rendered our cluster completely unavailable. It revealed that with spikes in throughput that are above the maximum tolerated throughput, the chance of blocking all of the the leader's threads increases. When this happens, the leader is less likely to send heartbeats in time. This triggers the leader election protocol. Leader election adds even more latency, while write requests are still ongoing, and causes further client requests to accumulate, ending up in a loop of leader elections and finally a complete crash of the system.

While the buffer mitigates this to a certain degree, it is also only able to cover a few spikes in throughput, but can not prevent this behavior in long-term throughput overruns.

## Benchmarking against Standalone ChronicleDB

### Write Throughput and Latency

// TODO consider Amdahls Law here for upfront theoretical considerations https://en.wikipedia.org/wiki/Amdahl%27s_law

### Querying and Aggregation

TODO only aggregates

### Horizonal Scalability

As we have seen so far, Raft introduces network latency and decreases the maximum throughput on a single stream, while this effect can be mitigated with horizontal scaling through partitioning and sharding. We compare serving multiple streams with ChronicleDB + Raft against standalone ChronicleDB.

TODO measuring multi-stream basis

\todo{measure!}

### Fault-Tolerance

The Raft protocol is formally proven (TODO reference to proof)...
It is not byzanthine-fault tolerant...

TODO any experimental setup to test fault tolerance anecdotably?

<!--
### Usability and Developer Experience
-->

## Profiling

We profiled our implementation to find the most CPU and I/O intense steps. We found that event serialization takes up a good amount of time, since we serialize events of a batch write sequentially on a single thread.

<!-- 

TODO image

TODO preserialization to estimate buffer size - need to measure again without this step

-->

## Comparison with other Replicated Event Stores and Time Series Databases

### Implementation Characteristics

### Write Throughput and Latency

TODO we haven't tested other storage systems ourselves, so we assume here the numbers of previous research and of the vendors themselves.

TigerBeetle: 1 million (TODO is this max, median? Find a paper)

"QuestDB shows a peak throughput of almost 1 mio rows/s, while InfluxDB only comes up with around 330k rows/s ."

TODO run tigerbeetles simulation service https://github.com/coilhq/tigerbeetle

\pagebreak

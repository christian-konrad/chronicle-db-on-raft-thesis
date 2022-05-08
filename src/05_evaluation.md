# Evaluation {#sec:evaluation}

\epigraph{I've always liked code that runs fast.}{--- \textup{Jeff Dean}}

<!--
Measurement results / analysis / discussion: 1/3
• whatever you have done, you must comment it, compare it to other systems,
evaluate it
• usually, adequate graphs help to show the benefits of your approach
• caution: each result/graph must be discussed! what’s the reason for this peak or why have you ovserved this effect
-->

This section covers the evaluation of the implementation of a replicated ChronicleDB event store with Raft presented in this paper. Especially, this section evaluates the cost of replication. Theoretically, the more the number of replicas, the higher the data availability; but the cost of replication (the detrimental in performance) increases at the same time. The challenge of the implementation is to achieve the optimal trade-off between the cost of replication and data access availability.

To evaluate this, the implementation is benchmarked against... as described in the following section.

## Setup for Comparison

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

## Throughput on Different Cluster Settings

TODO first describe expected latency decrease due to network roundtrip/maximum wide area delay of replica nodes (do some math here)

### Comparison of Cluster Sizes

### Comparison of Buffer Types and Sizes

### Throughput on a Distributed Remote Cluster

// TODO AWS stuff

## Benchmarking against Standalone ChronicleDB

### Write Throughput and Latency

// TODO consider Amdahls Law here for upfront theoretical considerations https://en.wikipedia.org/wiki/Amdahl%27s_law

### Querying and Aggregation

### Fault-Tolerance

The Raft protocol is formally proven (TODO reference to proof)...
It is not byzanthine-fault tolerant...

TODO any experimental setup to test fault tolerance anecdotably?

### Usability and Developer Experience

## Comparison with other Replicated Event Stores and Time Series Databases

### Implementation Characteristics

### Write Throughput and Latency

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

\pagebreak

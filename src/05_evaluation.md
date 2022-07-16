# Evaluation {#sec:evaluation}

\epigraph{I've always liked code that runs fast.}{--- \textup{Jeff Dean}}

<!--
Measurement results / analysis / discussion: 1/3
• whatever you have done, you must comment it, compare it to other systems,
evaluate it
• usually, adequate graphs help to show the benefits of your approach
• caution: each result/graph must be discussed! what’s the reason for this peak or why have you ovserved this effect
-->

In this work, an implementation of the Raft consensus protocol was proposed for replication of ChronicleDB, a high-throughput event store. It has been shown (TODO reference section) that Raft is strongly consistent, so a trade-off in performance is to be expected. In theory, the larger the number of replicas, the higher the data availability; however, at the same time, the cost of replication (which comes at the expense of performance) increases. This section addresses the evaluation of this implementation, specifically the cost of replication, by quantifying the costs and benefits that come with this approach. The challenge of the implementation is to find an optimal trade-off between the cost of replication and the availability of data.


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

TODO cite and compare with performance considerations of the raft dissertation https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

To evaluate this, the implementation is benchmarked against... as described in the following section.

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

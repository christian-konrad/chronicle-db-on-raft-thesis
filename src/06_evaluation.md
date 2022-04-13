# Evaluation {#sec:evaluation}

\epigraph{I've always liked code that runs fast.}{--- \textup{Jeff Dean}}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

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

### Comparison of Cluster Sizes

### Comparison of Buffer Types and Sizes

### Throughput on a Distributed Remote Cluster

// TODO AWS stuff

## Benchmarking against Standalone ChronicleDB

### Write Throughput and Latency

// TODO consider Amdahls Law here for upfront theoretical considerations https://en.wikipedia.org/wiki/Amdahl%27s_law

### Querying and Aggregation

### Usability and Developer Experience

## Comparison with other Replicated Event Stores and Time Series Databases

### Implementation Characteristics

### Write Throughput and Latency

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

\pagebreak

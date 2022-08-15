# Conclusion {#sec:conclusion}

\epigraph{All software architectures capture tradeoffs, and deciding between alternatives is about choosing which tradeoffs you can live with in a particular context.}{--- \textup{Jimmy Lin}}

This work covered the topic of replication for distributed event stores and comparable data stream management systems. We discussed the terms dependability, fault-tolerance and consistency and compared various replication protocols. Based on the potential use cases of a event stores and event processing systems that could be used in edge-to-cloud networks, we found that strong consistency satisfies all cases, even if it increases the total cost of replication, namely the write latency.

With strong consistency, we arrive somehow at a one-size-fits-all solution. It works in every case, but not perfectly. The trade-offs in latency and availability may make this approach not suitable enough for extremely high-throughput cases without further optimizations, but there are multiple other strategies to mitigate this, such as partitioning, which we introduced in our implementation. It may sound pessimisticly, but you can't have it all: the CAP and PACELC theorem have proven that in theory, and many implementations have shown that in practice. But actually, it is not that bad at all: even if there will always be a decision to be made between generalized vs. specialized implementations, both offer better dependability and scalability properties than a single-node deployment may ever be possible to serve.

As a result of our research, we presented ChronicleDB on a Raft, a Raft-based replication layer for the high-performance ChronicleDB event store. We have shown that we can provide a replication layer that allows us to deploy ChronicleDB on a distributed cluster, while providing strong consistency to clients—to some degree even for out-of-order events. We evaluated ChronicleDB in regards of the expected trade-offs. We have run all our evaluations on a single stream only. Both the median and maximum throughput for a single stream is significantly lower (almost 3 times for median and 5 times for maximum throughput) compared to standalone ChronicleDB on the same machine. We expect higher throughputs when writing to multiple streams, since this would leverage the write load distribution of Multi-Raft. With such horizontal scaling, we would expect to outperform standalone ChronicleDB. Since our prototype implementation is far from being optimized, we expect that a production-ready distributed ChronicleDB yields better performance, even on a per-stream level. In addition, running ChronicleDB on a distributed cluster increases hardware and infrastructure costs, but this is negligible compared to the benefits in achieving fault tolerance and availability.
 
<!--
Also write a discussion:
Interpretations: what do the results mean?
Implications: why do the results matter?
Limitations: what can’t the results tell us?
Recommendations: what practical actions or scientific studies should follow?


Summarise your key findings
Start this chapter by reiterating your problem statement and research questions and concisely summarising your major findings. Don’t just repeat all the data you have already reported – aim for a clear statement of the overall result that directly answers your main research question. This should be no more than one paragraph.

Examples
The results indicate that…
The study demonstrates a correlation between…
The analysis confirms…
The data suggests that…

https://www.scribbr.co.uk/thesis-dissertation/discussion/
https://www.scribbr.co.uk/thesis-dissertation/conclusion/
-->



## Recommendations and Future Work

\epigraph{\hfill Strive for progress, not perfection.}{}

<!--
We refer to Gotsman et al. [@gotsman2016cause] to help deciding for a consistency model. If low latency is important, we strongly advice system architects to think about their event design and apply practices of _domain-driven-design_ (DDD), such as breaking down their application to trivial facts (domain events) and derived aggregates and categorizing them as monotonic or non-monotonic, as described in subsection [@sec:consistency-decisions]. Afterwards, the appropriate consistency model can be decided upon, which further guarantees safety and liveness for the adapted system design. If latency is not that important or can be mitigated in other ways through sharding and other practices, we recommend opting for strong consistency, as this ensures safety in any case and allows starting with a simple system design that can be better thought out afterwards. Regarding out-of-order events, they must be either disallowed to provide strong consistency, or the consistency level must be explicitly flagged as eventual consistency, even if a strongly consistent replication protocol is used. We also recommend to look at time-bound partial consistency, as it can help with out-of-order events. We hope that this work will serve as a guidance here.
-->

We have noticed that our solution leads to a significant performance drop. Since we did not implement any optimizations other than a simple buffer, this is in line with our expectations. We have discovered possible optimizations and recommend system developers to take them into account when implementing a production-ready distributed event buffer.

### Consistency Considerations

Some related data stores such as InfluxDB show that strongly consistent replication of all data with a consensus protocol can be impractical and slows down the system, at least if no further optimizations are made in the replication layer. Others, such as TigerBeetle, have demonstrated that this can still be done. We refer to sections [@sec:cost-of-replication] and [@sec:user-decide-conclusion] for further considerations on consistency. The best compromise we can think of at the moment is to allow multiple levels of consistency to coexist in the database, and to let the user decide on the consistency model a per-stream basis.

There is a few recent literature with interesting approaches to state machine replication for streams, such as building the replication layer itself on top of a stream processing engine [@lawniczak2021stream]. We recommend to keep track of the literature and research here, since this is a relatively young area.

### Further Optimizations

In this final section, we outline some of the possible optimizations to Raft on the one hand, and to distributed event stores on the other. Throughout this work, we pointed to them in the discussions of consistency models, system design and our implementation.

- Sharding is essential to allow for horizontal scalability, either by time splits, on the current split, or both.
- Support of auto-scaling is crucial for production-ready systems, especially those running in containerized environments, such as Kubernetes.
- Monitoring throughput and stream convergence behaviour allows to continously calculate the maximum tolerated throughput that does not result in a negative convergence rate. Since negative convergence causes random leader elections, selected clients should receive push-backs in this case to prevent the system from becoming unavailable for all clients.
- Strong consistency is very expensive when running geo-replicated systems. We recommend to run an event processing agent between geo-replicated systems if strong consistency is not neccessary across the globe.
- There is a multitude of raft optimizations that we presented in section [@sec:raft-extensions], that can increase the total throughput, but may violate the understandability property of Raft. Another paper also tries to map popular optimizations of Paxos to Raft [@wang2019parallels].
- We use the naive implementation of the Raft log, provided by Apache Ratis, which is extremely slow. The Ratis team recommends that you implement your own log, and so do we. We proposed a log that only consists of pointers to the event store, to support the "The log is the database" philosophy again.
- We have not implemented a snapshotting mechanism for the event store state machine and therefore strongly recommend that this should be done.
- Our buffer is a simple solution, but it violates atomicity of Raft log entries. This should be done properly.
- We recommend to introduce learner roles or something similar to put load of from the leader. With increasing load, the leader is more likely to delay the delivery of heartbeats. We experienced in our evaluation, that with spikes in throughput that are above the maximum tolerated throughput, the chance of blocking all of the the leader's threads increases. Once this happens, the leader won't send heartbeats anymore in time, which triggers leader election. By moving the read load from the leader, the chance for this to happen decreases at least by some degrees.
- We can also imagine a dedicated role that serves continuous queries.
- But first, distributed queries must be implemented, since our prototype does not provide query processing at all.

\pagebreak

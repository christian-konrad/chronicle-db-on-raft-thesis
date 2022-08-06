# Protocol Decision and Implementation {#sec:implementation}

<!-- DONE -->

In the previous chapter, we introduced the reader to the realm of replication, explained the various properties of a replicated distributed system such as reliability, fault tolerance, consistency, and latency, and articulated a rationale for why a distributed database should be run with replicated data. We then discussed various consistency levels and replication protocols and provided guidance on how to decide on the appropriate model and protocol. We then looked at event stores and the requirements for them and introduced ChronicleDB, the event store we want to move to a distributed setting.

However, we have not yet provided explicit guidance on which consistency model and replication protocol is best suited for ChronicleDB under specific circumstances and use cases. Therefore, in this chapter we will demonstrate such guidance in three steps:

- First, we look at existing implementations of replication protocols for data-intensive applications and how they differ, focusing on event stores and other systems comparable to ChronicleDB.
- Second, we will take a closer look at the requirements of a potentially distributed ChronicleDB and decide on a consistency model and replication protocol, pointing out the trade-offs and how to mitigate them.
- Finally, we will describe the system design and architecture of a solution and discuss our demo implementation of such a replicated event store: ChronicleDB on a raft.

In the remaining chapters, we will then evaluate our implementation, compare the trade-offs with those of popular and comparable distributed database systems, and conclude by presenting the key learnings of this evaluation and pointing to possible future work.

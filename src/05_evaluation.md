# Evaluation {#sec:evaluation}

\epigraph{\hfill I've always liked code that runs fast.}{--- \textup{Jeff Dean}}

<!--
Measurement results / analysis / discussion: 1/3
• whatever you have done, you must comment it, compare it to other systems,
evaluate it
• usually, adequate graphs help to show the benefits of your approach
• caution: each result/graph must be discussed! what’s the reason for this peak or why have you ovserved this effect
-->

In this work, an implementation of the Raft consensus protocol was proposed for replication of ChronicleDB, a high-throughput event store. Since Raft is strongly consistent, a trade-off in performance is to be expected (see section [@sec:raft-costs]). In theory, the larger the number of replicas, the higher the data availability; however, at the same time, the cost of replication (which comes at the expense of performance) increases. In this section, we present the results from our evaluation of the prototype implementation, specifically the cost of replication, by quantifying the costs and benefits that come with this approach. The challenge of the implementation is to find an optimal trade-off between the cost of replication and the availability and consistency of data.

## Evaluation Goals

The main goal of this work is to test the additional cost of our techniques in terms of increased latency and reduced throughput in the system. We want to identify potential bottlenecks and ways to improve the prototype without sacrificing consistency. We evaluate different cluster settings and compare them with the standalone ChronicleDB to describe the impact on performance.

<!-- 
TODO measure:
* Network partition case
* Performance boost on horizontal scale (= partitioning, feeding in data in multiple groups)
* Read performance
-->

<!-- TODO use evaluation style of http://www.diva-portal.org/smash/get/diva2:24228/FULLTEXT01.pdf -->

## General Performance Considerations {#sec:general-performance}

Before running our evaluation, we set some expectations based on roundtrip and I/O considerations of Multi-Raft. Raft introduces at least one round-trip to a quorum of the nodes with the least delay when replicating a single log entry, and introduces additional I/O for writing log entries to disk—and probably less efficient in Ratis then in ChronicleDB itself. From what we've learned from other distributed event-store-alike systems, the throughput with strong consistency for a single, unpartitioned stream ranges from $\sim$ 330,000 operations/sec for InfluxDB, partially running on Raft + Primary-Copy, to $\sim$ 1,000,000 operations/s for TigerBeetle on Viewstamped Replication (while this actually accounted for transactions, not just single events), running on comparable setups (cf. subchapter [@sec:previous-work]). We also expect a linear correlation between the number of nodes in a cluster and decreased write throughput, based on our observations.

## Limitations

##### Read Saturated Systems.

In our experimental setup, we only measured write performance in isolation. We did not measured write performance in read-saturated systems—which comes closer to a real-world usage example.

##### Recent Improvements.

We already identified bottlenecks in our implementation (cluster management and application logic servers too tightly coupled, redundant serialization, and an inefficient Raft log implementation) and started working on them. To the time of this writing, we didn't finished our work, though. We would expect these changes to improve the throughput by a significant number, but we can not validate this yet.

##### Multi-Partition Performance.

With multiple partitions, the write throughput is expected to increase because write load can be distributed across leaders. Unfortunately, at the time of this writing, we did not managed to finish our work on instantiating multiple partitions and streams in our prototype, and therefore, we can not validate this.

##### Upscaling Cluster Machines.

We evaluated throughput only on very cheap machines on AWS. We did not measure the impact of upscaling these machines. The effect of upscaling should be evaluated to compare the benefit in throughput with the additional cost of hardware (multiplicated with every replica).

## Setup for Comparison

We run our evaluation application (see section [@sec:test-application]) on two different machine setups. The first setup is a deployment on a local machine, and the second setup is a deployment on seperate machines in the same availability zone.

\begin{table}[H]
    \caption{Setup for evaluation on a local machine}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X} 
        \toprule
        Code & ChronicleDB Cloud v0.0.1-alpha\footnote{https://www.doi.org/10.5281/zenodo.6991873} \\
        OS & Microsoft Windows 10 Professional 64 Bit \\
        CPU & Intel Core i7-10510U (4 Cores) \\
        Disk & SSD (PCIe NVMe) \\
        Memory & 32 GB DDR4 SDRAM \\
        Network & Protocol Buffers over Localhost Loopback \\
        \bottomrule
    \end{tabularx}
    \label{table:local-evaluation-setup}
\end{table}

\begin{table}[H]
    \caption{Setup for evaluation on a remote cluster}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X} 
        \toprule
        Code & ChronicleDB Cloud v0.0.1-alpha\footnote{https://www.doi.org/10.5281/zenodo.6991873} \\
        OS & Amazon Linux 4.14.262-200.489.amzn2.x86\_64 \\
        CPU & 1 vCPU \\
        Disk & SSD \\
        Memory & 2 GB \\
        Network & Protocol Buffers over HTTP/2, $\sim$ 500 MB/s up-/download \\
        Region & eu-central-1 \\
        \bottomrule
    \end{tabularx}
    \label{table:remote-evaluation-setup}
\end{table}

We also run these two cluster setups with different replication factors and buffer sizes.

<!--
- TODO not just analyze throughput on a single stream. This gets slow indeed.
- also evaluate horizontal scalability! Which means multiple different streams (with partitioning).
    - Expected: Faster than standalone (mention here standalone could benefit from multithreaded/concurrency, but only to a certain limit (CPU cores, shared memory etc.)) 
    - TODO reference/citate literature on this
-->

##### Simulation Data.

We simulate stock price data by creating random events of the following schema in advance, holding them in memory on one node and then inserting them into the `ReplicatedEventStore`.

```
SYMBOL: String,
SECURITYTYPE: Integer,
LASTTRADEPRICE: Float
```

We can run one trial of such an experiment by first creating a stream partition for the target schema and then calling the REST endpoint that starts the evaluation run. This evaluation service measures and returns us the total runtime of inserting the events. The whole experimental setup and evaluation can be found and repeated by yourself in the Jupyter notebook in the appendix.

We run all our trials on a single stream only. We expect higher throughputs when writing to multiple streams, since this would leverage the write load balancing of Multi-Raft.

## Dependability

Since we use Apache Ratis on the replication layer, most of the dependability results are based on the Ratis implementation. Unfortunately, Apache Ratis does not come with a validation of liveness and safety, so we assume that it follows the Raft $\textrm{TLA}^{+}$ spec, but we also expect bugs to appear that may not be easy to detect. Therefore, our conclusion on each dependability property is: 

- **Availability**: Distributed ChronicleDB is available as long as a quorum of nodes is available.
- **Reliability**: As long as the maximum throughput has not been exceeded for an extended period of time, the cluster operated reliably. With throughput spikes for a longer period of time, the cluster crashed and had to be restarted (see section [@sec:stress-test]).
- **Safety**: Raft provides strong consistency, so as long as Ratis follows the full specification, all nodes will follow the same operation order, therefore exposing equal event stores. We did not implemented a byzantine-fault tolerant Raft, so tampering is still a possible thread. Therefore, we do not recommend to allow distibuted ChronicleDB to run without permissions.
- **Integrity**: As long as Ratis follows the full Raft specification, integrity should not be violated.
- **Maintainability**: Our modular architecture generally allows to maintain a state machine instance without disrupting other running instances.

## Ratis Performance

Running our prototype without a buffer has shown that Apache Ratis introduces a huge decrease in throughput. On a local machine with three virtual nodes (see table \ref{table:local-evaluation-setup}), our implementation of Ratis using the naive default log implementation yields a maximum throughput of 250 operations/s with a median of 95 operations/s. As the Ratis team claims, the naive log should not be used for write-intense applications, and custom implementation is mandatory to serve high throughputs (see section [@sec:library-decision]).

## Throughput on Different Cluster Settings

To evaluate our prototype, we measure the throughput when the system is saturated with writes.

### Comparison of Cluster Sizes

We compare different cluster sizes on a local machine (see table \ref{table:local-evaluation-setup}). So far, this result must be taken with a grain of salt, since we did this comparison on a local machine only, which quickly hits I/O boundaries of the machine. 

In figure \ref{fig:cluster-sizes-comparison}, we plot our results of the performance comparison of clusters with different replication factors.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/cluster-sizes-comparison.pdf}
  \caption[Cluster size performance comparison]{Comparison of the median and maximum performance of different cluster sizes for a single stream on a local machine.}
  \label{fig:cluster-sizes-comparison}
\end{figure}

We expected a linear correlation between the number of nodes in a cluster and decreased write throughput, similar to other distributed storage systems. We actually observe a slightly logarithmic trend, which we assume is the case because we approached maximum I/O on our single machine. Since we did not evaluate this on clusters with actual different physical machines, we can not validate this hypothesis here. 

### Comparison of Buffer Sizes

Without a buffer, writing every event to the Raft log slows down the system tremendously, due to increased I/O and especially because the default log in Ratis requires a log entry to be written before it is sent to other nodes, effectively blocking the whole system on every single operation, as we already mentioned.

We compared different buffer sizes of 1KB, 10 KB, 100 KB and 1 MB. Unsurprisingly, a larger buffer yields better performance, but from our results in figure \ref{fig:buffer-size-comparison-throughput} it seems that it approaches a certain limit, that is far from the performance that we achieve with standalone ChronicleDB. We show the different buffer sizes on a log scale, and the median and max throughput on a linear scale.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/buffer-size-comparison-throughput.pdf}
  \caption[Comparison of throughput of different buffer sizes]{Comparison of the throughput of different buffer sizes with standalone ChronicleDB as a benchmark}
  \label{fig:buffer-size-comparison-throughput}
\end{figure}

To put our results in a different perspective, we show in figure \ref{fig:median-time-to-ingest} the median time to ingest 1,000,000 events.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.6\textwidth]{images/median-time-to-ingest.pdf}
  \caption[Comparison of ingestion time for different buffer sizes]{Median time to ingest 1 mio events for different buffer sizes}
  \label{fig:median-time-to-ingest}
\end{figure}

The boxplots in figure \ref{fig:buffer-boxplots} in addition show how our distributed ChronicleDB prototype competes against a standalone deployment in both median and maximum throughput. While the median throughputs are relatively close, the maximum throughputs differ significantly. We explain this with our observations in section [@sec:stress-test].

\begin{figure}[H]
  \centering
  \includegraphics[width=0.7\textwidth]{images/buffer-boxplots.pdf}
  \caption[Boxplots of throughput of different buffer sizes]{Comparison of the throughput of different buffer sizes with standalone ChronicleDB as box plots}
  \label{fig:buffer-boxplots}
\end{figure}

The size of a log entry in Ratis is limited. We tried to increase it, but somehow failed. So, the maximum size of a log entry is what limits our maximum buffer size, since we do not extract single events into single log entries and send them in batches yet (see section [@sec:buffered-inserts]).

### Throughput on a Distributed Remote Cluster

We evaluated also on a remote setup (see table \ref{table:remote-evaluation-setup}), which yields slighlty better results compared to those on our local setup. Since we used very cheap machines on AWS to run this evaluation, compared to our local machine, we interpret these results as a success; however, we can not validate if upscaling single machines in the remote cluster will improve the performance since we did not measure that. If upscaling would not improve throughput, the bottleneck is clearly in the replication layer.

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/minmax-aws-local.pdf}
  \caption[Performance of local vs. remote cluster]{Performance of local vs. remote cluster}
  \label{fig:minmax-aws-local}
\end{figure}

## Exceeding the Upper Throughput Threshold {#sec:stress-test}

We stress-tested the cluster by increasing the write load in a way that the mean throughput exceeds the maximum throughput for a longer period of time.

The stress-test rendered our cluster completely unavailable. It revealed that with spikes in throughput that are above the maximum tolerated throughput, the chance of blocking all of the the leader's threads increases. When this happens, the leader is less likely to send heartbeats in time. This triggers the leader election protocol. Leader election adds even more latency, while write requests are still ongoing, and causes further client requests to accumulate, ending up in a loop of leader elections and finally a complete crash of the system.

While the buffer mitigates this to a certain degree, it is also only able to cover a few spikes in throughput, but can not prevent this behavior in long-term throughput overruns.

## Benchmarking against Standalone ChronicleDB

As shown in figures \ref{fig:buffer-size-comparison-throughput} and \ref{fig:buffer-boxplots}, The performance of standalone ChronicleDB for a single stream is superior to that of the distributed solution. With a replication factor of 3 and a buffer size of 1 MB, the median throughput is up to 3 times higher on standalone ChronicleDB, and the maximum throughput is more than 5 times higher. We assume that this can be outweighted quikcly as soon as multiple streams are served, because distributed ChronicleDB can balance the write load on multiple leader nodes. We even do not consider this to be bad, since we haven't implemented any optimizations yet, such as a pointer-only Raft log with concurrent access, and this results still work in a edge-cloud setting, where sensors run the embedded, high-throughput version of ChronicleDB, and only aggregated events will be written to the distributed deployment(s).

### Querying and Aggregation

We implemented the aggregate interface in our distributed ChronicleDB prototype. We continuously ran aggregates with our evaluation application. We did not see any significant differences in read performance compared to standalone ChronicleDB.

We have not implemented the query interface (see section [@sec:system-design-limitations]), so we can not reason about this here.

## Profiling

We profiled our implementation to find the most CPU and I/O intense steps. We found that event serialization takes up a good amount of time, since we serialize events of a batch write sequentially on a single thread. We also use parts of the event serializer to estimate the size of an event before writing it into the buffer. We could omit this step by allowing the buffer size to be configured by maximum event count instead of bytes, but since events can have different schemas and variable sizes (when strings are used), we need to estimate the buffer size to not exceed the maximum size of a Ratis Raft log entry.

## Comparison with other Distributed Event Stores and Time Series Databases

Compared to other distributed event stores and similar systems, we achieve similar results on the lower end of the range (cf. section [@sec:general-performance]). Since we did not implemented any further optimizations yet, we assume that distributed ChronicleDB is actually capable of outperforming this systems. We can learn from the implementation of the replication layers of those competing systems, and standalone ChronicleDB outperforms most of other standalone systems, so this should be possible.

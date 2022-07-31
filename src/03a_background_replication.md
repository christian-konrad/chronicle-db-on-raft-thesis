## Replication {#sec:why-replication}

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport}}
<!-- [@milojicic2002discussion] -->

<!-- ### why replication is neccessary, and what it is-->

This section provides the reader with an overview of the topic of _distributed database systems_ (DDBS), defines the term _replication_, explains why replication is crucial for many of those systems, discusses various replication protocols, and concludes with a literature review of recent research in this area. The goal of this section is to provide the reader with a thorough understanding of what replication is, why, where, and when it is needed, and how replication protocols work.

\todo{Glossary?}

A _distributed system_ is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed]. To achieve this behavior, such a system needs to offer certain levels of _availability_, _consistency_, _scalability_ and _fault-tolerance_; therefore, many modern distributed systems rely heavily on replicated data stores that maintain multiple replicas of shared data [@liu2013replication]. Depending on the replication protocol, the data is accessible to clients at any of the replicas, and these replicas communicate changes to each other using message passing.

Thus, a distributed database system is a distributed system that provides read and write access to data. Replication in these databases takes place to varying degrees: from simple replication of metadata and current system state to full replication of all payload data, depending on the requirements and characteristics of the system.

Many modern applications, especially those served from the cloud, are inherently distributed. With edge computing, such applications can be distributed not only to nodes in one or more clusters serving many clients, but also to all of those clients participating in the edge-cloud network. Users of cloud applications expect the same experience independent of their geographic region and also regardless of the amount of data they produce and consume.

In complex distributed systems, there are multiple interoperating replicated _(micro)services_ with different availability and consistency characteristics. Ensuring that these systems and services can reliably work together is the topic of container orchestration (e.g., through Kubernetes [@kubernetes2022github]) and service composition [@nikoo2020survey; @camunda2022zeebe]—which need to be replicated, too—which we will not discuss in this work.

Example use cases for replication in distributed systems (additionally to those mentioned in the [Introduction](#sec:introduction)) include large-scale software-as-a-service applications with data replicated across data centers in geographically distinct locations [@mkandla2021evaluation], applications for mobile clients that keep replicas near the user's location to support fast and efficient access [@silva2022geo], key-value stores that act as bookkeepers for shared cluster metadata [@etcd2022], highly available domain name systems (DNS) [@bi2020craft], blockchains and all their various use cases (see [Blockchain Consensus Protocols](#sec:blockchain-consensus)), or social networks where user content is to be distributed cost-effectively to millions of other users [@khalajzadeh2017cost], to name a few.

The following subsections describe the fundamental concepts of _dependable_ distributed systems, as well as use cases for replication and the challenges in these particular cases, before outlining relevant replication protocols.

### Horizontal Scalability {#sec:scalability}

Standalone applications can benefit from _scaling up_ the hardware and leveraging concurrency techniques/multithreading behavior (in this case we speak of _vertical scalability_), but only to a certain physical hardware limit (such as CPU cores, network cards or shared memory). Approaching this limit, these applications and services can no longer be scaled economically or at all, as the hardware costs increase dramatically. In addition, applications running on only one single node[^node-process] pose the risk of a single point of failure, which we want to counter with replication.

[^node-process]: Note that in this work, we use the terms _node_ and _process_ interchangeably when talking about replication.

To scale beyond this limit, a system must be designed to be _horizontally scalable_, that is, to be distributable across multiple computing nodes/servers (also known as to _scale out_). Ideally, the amount of nodes scales with the amount of users, but to achieve this, certain decisions must be made regarding [consistency models](#sec:consistency) and [partitioning](#sec:partitioning). Legacy vertically scalable applications cannot be made horizontally scalable without rethinking the system design, which includes replication and message passing protocols between the nodes to which the application data is distributed.

With vertical scaling, when you host a data store on a single server and it becomes too large to be handled efficiently, you identify the hardware bottlenecks and upgrade. With horizontal scaling instead, a new node is added and the data is partitioned and split between the old and new nodes.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.85\textwidth]{images/scaling.pdf}
  \caption[Vertical vs. horizontal scaling]{Vertical vs. horizontal scaling. a) With vertical scaling, existing machines are scaled up by upgrading. b) With horizontal scaling, more machines are added to handle and balance increased loads.}
  \label{fig:scaling}
\end{figure}

With an increasing number of users served, the number of nodes in a horizontally scalable system grows. In the ideal case, the number of nodes required to keep the perceived performance for users constant increases linearly with the number of distinct data entities served (such as tables or data streams with distinct schemas) [@williams2004web]; in other words, the transaction rate grows linearly with the computational power of the cluster. This is not neccessarily the case when a increasing number of users access the same stream/table or inter-node/inter-partition transactions are executed. In the latter case, vertical scaling or advanced partitioning techniques [@aaqib2019efficient] still come handy. In practise, many applications achieve near-linear scalability.

\todo{NTH: show chart and math for near-linear scalability if time}

\todo{Charts: vertical vs horicontal scalability (i.e. the zeebe one)}

Opting in for horizontal scaling is beneficial for large-scale cloud and edge applications as it is easier to add new nodes to a system instead of scaling up existing ones, even if the initial costs are higher as the infrastructure and algorithms must be implemented in first place.

The system design that enables horizontal scalability is also a key requirement for replication. To achieve fault tolerance and availability through replication, it is required to store replicas of a dataset on multiple nodes, therefore the book keeping and message passing infrastructure that is neccessary for partitioning is also a basic requirement for replication protocols [@liu2013replication].

### Safety and Reliability {#sec:safety-reliability}

\epigraph{Anything that can go wrong will go wrong.}{--- \textup{Murphy's Law}}

Modern real-time distributed systems are expected to be reliable, i.e., to function as expected without interruption, and to be safe, i.e., not to cause catastrophic accidents even if a subsystem misbehaves. For a distributed system to be both reliable and secure, it must be designed to be _fault-tolerant_, i.e., the application must be able to cope with node failures without interrupting service. In addition, it must also be able to withstand faults without operating incorrectly, i.e., responding to user requests with erroneous or malicious content. In large distributed systems, faults will happen - they are inevitable. Therefore, fault-tolerance is the realization and acknowledgement that there will be faults in a system. There is no system that is 100 % fault-tolerant. A system that can tolerate at least $k$ faulty nodes is called $k$-_fault-tolerant_.

\todo{K-fault-tolerance formatted as a definition?}

To understand how a system can be designed to be both reliable and secure, we review the definitions of reliability and security [@mulazzani1985reliability; @farooq2012metrics]:

\todo{If sufficient time, use actual definition formatting for things like this instead of just paragraphs}

\paragraph{Reliability.} The _reliability_ of a system is the probability that its functions will execute successfully under a given set of environmental conditions (operational profile[^operational-profile]) and over a given time period $t$, denoted by $R(t)$. Here $R(t) = P(T > t), t \geq 0$ where $T$ is a random variable denoting the time to failure. This function is also known as the _survival function_.

[^operational-profile]: An operational profile is a quantitative characterization of how a system will be operated by actual users [@musa1993operational]. Different users using the same system in different ways may experience different levels of reliability. An operational profile shows how to increase productivity and reliability by allowing us to quickly find the faults that impact the system reliability mostly and by finding the right test coverage.

There are also other dimensions of interest to express the reliability of a system:

\begin{enumerate}[label=(\arabic*)]
  \item The probability of failure-free operation over a specified time interval, as in above definition.
  \item The expected duration of failure-free operation.
  \item The expected number of failures per time interval (failure intensity).
\end{enumerate}

The second can be measured using the _Mean Time Between Failures_ (MTBF). The MTBF is a common and important measure and is especially critical for consensus protocols, a particular class of replication protocols, since the MTBF of a single node must be estimated in advance when defining the protocol's timing criteria,  which will be shown later in [Section 1.2](#sec:raft) when we describe the replication protocol of choice of this work. The MTBF itself is a combined metric: it is the sum of the _Mean Time To Failure_ (MTTF) and the _Mean Time To Repair_ (MTTR).

The MTTF describes the mean time of uninterrupted _uptime_ of a system where it is operational, therefore the average time between a succesful system start and a failure:

\begin{equation}\label{eq:mttf}
\textrm{MTTF} \coloneqq \sum_i \frac{t_{\textrm{down},i} - t_{\textrm{up},i}}{n} = \sum_i \frac{u_i}{n}
\end{equation}

where $t_{\textrm{down}}$ is the start of the downtime after a failure, $t_{\textrm{up}}$ the start of the uptime before that failure, $u_i$ the current uptime period and $n$ the number of failures.

This only describes the uptime: the _downtime_ or repair time, including failure detection, reboot, error fixing and network reconfigurations are described by the MTTR. In some literature, the average amount of time it takes to detect a failure is additionally extracted as a dedicated metric, the _Mean Time To Detect_ (MTTD)[^MRDP]. 

[^MRDP]: Another metric, often used in manufacturing, is the _Mean Related Downtime for Preventive Maintenance_ (MRDP), which describes a planned time of unavailability to conduct preventive actions that will actually reduce the net unavailability, as they reduce the risk of failures. In distributed systems, there are strategies for _hot updates_ that allows to conduct preventive maintenance node by node without shutting down the whole system, as long as at least one replica set of nodes remains available, while other nodes are updating. This is limited to updates without breaking changes that disrupt interoperability of nodes. Modern messaging protocols like _Protocol Buffers_ (which are used in this work) support this by providing schema-safety with both backward and forward compatibility [@google2022protobuf].

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/mtbf_relation.pdf}
  \caption[Relationship between the MTBF and other reliability metrics]{Relationship between the Mean Time Between Failure (MTBF) and other reliability metrics}
  \label{fig:mtbf-relation}
\end{figure}

Actually, the MTBF can also be expressed as an integral over the reliability function $R(t)$ [@birolini2013reliability], which illustrates the relation between the two metrics:

$$ \textrm{MTBF} = \int_{0}^{\infty} R(t) dt $$

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/mtbf-r-curve.pdf}
  \caption[A curve for a typical $R$ function]{A curve for a typical $R$ function, illustrating the relation between the $R$ and MTBF metric}
  \label{fig:mtbf-r-curve}
\end{figure}

One of the most important design techniques to achieve reliability, both in hardware and software, is redundancy [@cristian1991understanding]. There are two main patterns to achieve redundancy: retry and replication. Retry is _redundancy in time_, while replication is _redundancy in space_. Using a retry strategy, a _failure detector_ oftentimes is just a timeout. In this case, the MTTD is the timeout interval while the MTTR is the retry time. In replication, timeouts are also used to detect dead nodes (assuming a fail-stop failure model, as described in the following paragraphs) so they can be rebooted in the background or the network can be reconfigured (by the network itself or an external book keeper or orchestrator), commonly covered by the rules of the replication protocol and oftentimes without having a perceivable impact on reliability and availability for the user.

\paragraph{Safety.} The safety of a system is the probability that no undesirable behavior or even _catastrophic accidents_ will occur during system operation over a specified period of time. Safety looks at the consequences and possible accidents, and how to deal with it.

\todo{Explain what a catastrophic accident is}

Safety is an important requirement especially for critical infrastructure applications. Safety means that the system behavior is as expected, and no faulty or even malicious results are returned.

A high degree of reliability, while necessary, is not sufficient to ensure safety. In fact, there can even be a trade-off between safety and reliability. To understand this, we need a different way to model reliability and safety. We look at the probabilities of three possible types of responses to a client request to the system:

- The probability of a _good response_ $p_g$, which is as intended based on the problem (not only to the specification, as it could be faulty) and delivers a correct value in a timely manner. 
- The probability of a _faulty response_ $p_f$, meaning that the system responded but returned an erroneous or malicious result instead of the intended result (i.e. a _byzanthine fault_, as described later in [the subsection on Types of Possible Faults](#sec:possible-faults)).
- The probability of _no response_ $p_n$, due to a system crash or under other _fail-stop_ conditions, as described later.

With these probabilities, we can then have a simplified model for reliability:

$$ R \coloneqq 1 - p_n - a \cdot p_f, \quad 0 \leq a \leq 1 $$

where $a$ is the fault detection factor, denoting the rate of faulty results detected by the system (and resulting in a fail-stop, i.e. the faulty results are not returned). A system, where faulty responses are not detected, can be therefore perceived as more reliable (as the user may not be able to dinstinguish faulty and intended results), while this can pose a severe safety threat. Take for example a banking transaction. With a bank transfer, safety is more important than reliability, which means in case of a fault, no money should be transferred at all, instead of sending the wrong amount of money or even to the wrong account (or to the sneaky hacker trying to steal your money).

On the other hand, safety can then be modeled as

$$ S \coloneqq 1 - b \cdot p_f - c \cdot p_n, \quad 0 \leq b,c \leq 1 $$

where $b$ and $c$ are the probabilities that a faulty and no-response event, respectively, will cause undesired behavior or even disastrous consequences to the user. Oftentimes, we have systems where $b \gg c \approx 0$, which means that the safety depends on correct, as-intended results, such as in the bank transfer example.

This definitions of safety and reliability are related to the definition of the _safety and liveness properties_ in the concurrency literature. If a system applies one of these properties, then it is true for every possible execution in the system:

- **Safety**: Something bad will not occur, 
- **Liveness**: Something good will _eventually_ occur.

#### Types of Possible Faults {#sec:possible-faults}

\epigraph{To a first approximation, we can say that accidents are almost always the result of incorrect estimates of the likelihood of one or more things.}{--- \textup{C. Michael Holloway, NASA}}

On the one hand, faults[^faults] can be categorized by the nature of their timing:

- **Transient faults**: These faults occur once and then disappear. For example, a network request that timed out but succeeded after a retry.
- **Intermittent faults**: These faults occur many times in an irregular fashion and are hard to repeat reliably. Often, several different events must occur at the same time to contribute to and cause such a fault, so that the fault appears random; as a result, it is more complicated to perform a root cause analysis for such faults.
- **Permanent faults**: These faults are persistent and either make the system halt (fail-stop) or cause ongoing faulty behavior of the system. These faults persist until they are actively fixed, but since they are permanent, their detection is more straightforward than that of intermittent errors.

Studies have shown that intermittent and transient failures cause a significant portion of downtime in large scale distributed systems [@candea2003crash].

[^faults]: A _fault_ is the initial root cause, including machine and network problems and software bugs. A _failure_ is the loss of a system service due to a fault that is not properly handled [@farooq2012metrics]. The probability of a fault manifesting itself as a failure is not uniform: only a few faults actually cause a system failure, and the actual system downtime is caused by an even smaller group of faults. When thinking about fault detection, it is therefore important to identify and focus on this small, but significant group of faults.

On the other hand, we need to categorize faults in such a way that we can decide which and how many faults a system should tolerate and how we want to implement this fault-tolerant behavior while still having a useful and maintainable system. No system can tolerate all kinds of faults (e.g. if all nodes crash permanently), so we need a model that describes the set of faults allowed. There are different types of such faults and two major models to describe them [@bracha1983resilient]:

\paragraph{Crash Failure/Fail-Stop.} Processes with fail-stop behaviour simply "die" on a fault, i.e. they stop participating in the protocol [@schlichting1983fail]. Such a process stops automatically in response to an internal fault even before the effects of this fault become visible. In asynchronous systems, there is no way to distinguish between a dead process and a merely slow process. In such a system, a process may even appear to have failed because the network connection to it is slow or partitioned. However, designing a system as a fail-stop system helps mitigate long-term faulty behavior and can improve the overall fault-tolerance, performance and usability by making a few assumptions [@candea2003crash]: one approach to assuming such a failure is to send and receive heartbeats and conclude that the absence of heartbeats within a certain period of time means that a process has died. False detections of processes that are thought to be dead but are in fact just slow are therefore possible but acceptable as long as they are reasonable in terms of performance. With a growing number of processes involved in a system, such failure detection approaches can become slower and  lead to more false assumptions, so more advanced error detection methods come into play, for example based on gossiping [@renesse1998gossip] or _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable].

Examples for faults causing such a fail-stop failure in a distributed system are Operating System (OS) crashes, application crashes, and hardware crashes. As shown before, intermittent and transient failures, if not handled properly, are responsible for a huge portion of downtime of systems. Because they can lead to unexpected behavior if not properly detected, most systems are designed to _fail gracefully_ in the event of such failures, e.g., through proper code design that always throws a runtime exception that is caught by the application runner and forces a reboot (or other strategies, such as a network reconfiguration in consensus protocols). Candea et al. suggest designing systems to be crash-only in the event of faults [@candea2003crash] as they have shown that a reboot can actually safe time, since faults are oftentimes resolved by a reboot, and the time to reboot (the MTTR) can be shorter than the downtime or slowdown of the system caused by the fault itself. In modern service-oriented approaches, a suitable strategy is to simply "throw away" a faulty node while at the same time reinitialising a new one, rather than directly mitigating the cause of the fault, especially for stateless services. In consensus protocols, shutting down a faulty node and initializing a fresh new node is effective as well because the new node's data can be initialized in the background using snapshotting without affecting the performance of the entire cluster, as we show later in the corresponding section. 

\paragraph{Byzantine Faults.} Instead of crashing, faulty processes can also send arbitrary or even malicious messages to other processes, containing contradictory or conflicting data. Even a bitwise failure in a network cable can cause such a fault. The effect of the fault becomes visible (while being hard to detect and to distinguish from an intended response) and, if not handled properly, can negatively impact the future behavior of the faulty system, resulting in a _byzantine failure_. Byzantine failures compromise the safety of a system far more, as these faults can result in silent data corruption leaving users with possibly incorrect results. A detector for such faults is difficult to create, especially since some faults can only be detected at all under certain circumstances. There is no generic approach to detect such faults in single-node systems. (TODO is this true? What about checksums?) Even worse, numerous security attacks can be modeled as Byzantine failures, such as censorship, freeloading or misdirection. Systems can be protected with _Byzantine fault tolerance_ (BFT) techniques (as we will show later in the [subsection on Consensus Protocols](#sec:consensus-protocols)) by _masking_ a limited number of Byzantine failures. This work focuses on _failure masking_, which makes a service fault-tolerant, while it is also worthwhile for the reader to look at _fault detection and correction_ techniques[^fault-detection]. 

[^fault-detection]: There is not much literature on the subject of byzantine fault detection. The best known approach is the PeerReview detection algorithm [@haeberlen2006case], which extends the Chandra–Toueg consensus algorithm mentioned earlier. A very recent approach that outperforms PeerReview for transactional databases is Scalar DL [@yamada2022scalar]. Due to some limitations of fault detection, especially in terms of safety characteristics, but also scalability, failure masking (i.e., byzantine fault-tolerance via consensus protocols) remains the most common and studied approach.

In a naive approach, under the assumption that all nodes process the same commands in the same order, a system of $n$ replicas is considered to be fault-tolerant (or $k$-_resilient_) if no more than $k$ replicas become faulty:

- Fail-stop failure: $n \geq k + 1$, as $k$ nodes can fail and one will still be working,
- Byzantine failure: $n \geq 2k + 1$, as while $k$ nodes generate false replies, the remaining $k+1$ nodes will still provide a majority vote.

This naive approach does not incorporate timing characteristics in messaging between the nodes of the system and the risk of network partitioning. To ensure that all nodes process the same commands in the same order, they need to reach consensus on which command to execute next. This is explained in detail in the [subsection on Consensus Protocols](#sec:consensus-protocols)), but here's the short version. $n$ nodes are needed to reach consensus in case of $k$ faulty replicas:

- Fail-stop failure: $n \geq 2k + 1$,
- Byzantine failure: $n \geq 3k + 1$.

This also shows that all fail-stop problems are in the space of byzantine problems, too, therefore, a $k$-byzantine-fault-tolerant system is automatically $k$-crash-fault-tolerant.

Note that not all authors agree to the binary model of fail-stop vs. byzantine faults, claiming that the fail-stop model is too simple to be sufficient to model the behavior of many systems, while the byzantine model is too generalistic, thus too far away from practical application, and so is byzantine fault-tolerance hard to achieve. At least two other fault models are subject of the literature: the _fail-stutter_ fault model [@arpaci2001fail] is an attempt to provide a middle ground model between these two extremes which also allows for _performance faults_, and the _silent-fail-stutter_ fault model tries to extend it furthermore [@kola2005faults]. Despit their usefulness, both models are not quite popular and research on replication protocols does not take those into account, therefore we won't pay too much attention on them in this work. The relation of all this different fault models is shown in figure \ref{fig:error-classes-venn}.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/error-classes-venn.pdf}
  \caption{Relation of the different fault models}
  \label{fig:error-classes-venn}
\end{figure}

#### Partition-Tolerance

In addition to the discussed types of faults, there is also the risk of _network partitioning_. There are various types of network partitions, including complete, partial and simplex partitions. In figure \ref{fig:error-classes-venn}, all three are shown for reference. In (a), we have a complete partitioning of the network, resulting in a split into two disconnected groups, also known as a _split brain_. In (b), not all nodes are affected by the partitioning, so there is at least one route between all nodes, which is called _partial partition_. In (c), there is a _simplex partition_, in which messages can only flow in one direction. Network partitioning events can be of a temporary or persistent nature, depending on the original fault that caused them, but also on the strategies used to resolve them. In large-scale distributed systems, network partitioning is to be expected. Particularly in massive-scale distributed systems like blockchains, network partitioning is part of the design.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/partitioning-all.pdf}
  \caption[Possible network partitioning types]{Possible network partitioning types. (a) Complete partition/split brain, (b) partial partition, (c) simplex partition.}
  \label{fig:partitioning-types}
\end{figure}

If not handled correctly, the partitioning can lead to Byzantine behavior once the partitioning is resolved again. Partitioning also compromises consistency, as the resulting subnetworks may serve different clients, aggregating different sets of data until they are reunited. Alquraan et al. found that network-partitioning faults lead to silent catastrophic failures, such as data loss, data corruption, data unavailability, and broken locks, while in 21 % of failures, the system remains in a persistent faulty state that persists even after partitioning is resolved [@alquraan2018analysis]. Those faults occur easily and frequently: in 2016, a partitioning happened once every two weeks at Google [@govindan2016evolve] and in 2011, 70% of the downtime of Microsoft's data centers where caused by network partitioning [@gill2011understanding]. Even partial partioning causes a large number of failures (due to bad system design), which may be suprising as there is still a functioning route for messages to pass through the network.

Partition-tolerance is in general a must-have for a distributed system. In the case of partitioning, users and clients expect the system to still be available and reliable, without compromising the safety by subsequent faulty behavior. The don't want to experience the interruption by the partitioning at all. For a replication protocol to be truly fault-tolerant for large-scale or geographically distributed systems, partitioning must be considered, as it is unavoidable, even partial partitioning as it has been shown. In consensus protocols, this is a mandatory requirement. Container orchestration services like Kubernetes help in resolving partitioning by _network reconfiguration_ if they detect node failures and partitioning, which must be taken into account be the consensus protocol.

#### Disaster Recovery {#sec:disaster-recovery}

Another type of failure that is important to address are _disasters_ in the data center. _Disaster recovery_ includes all actions taken when a primary system fails in such a way that it cannot be restored for some time, including the recovery of data and services at a secondary, surviving site. One important disaster recovery measure is _geo-replication_, which can be used to manage disasters of the type that render data centers unavailable due to, for example, a natural disaster such as a flood or earthquake. Geo-replication is discussed briefly in subsection [@sec:geo-replication]. Since disaster recovery is a complex subject area of its own that goes beyond the application of replication mechanisms, it is not discussed in detail in this thesis.

### High Availability and Dependability {#sec:availability}

To understand what _availability_ means, it is worth reviewing the definitions of availability and discussing them in the context of the other related concepts we have already addressed in the previous subsection, namely reliability and safety. These concepts are summarized under the term _dependability_. 

Dependability, as defined by Avizienis et al., can be described in several ways [@avizienis2004basic]: first, as the ability to provide services that can be justifiably trusted. A more precise definition is also given as the ability to avoid service failures that are more frequent and more severe than is acceptable. In addition to these definitions, dependability is better understood as an integrating concept that subsumes the following requirements:

- **Availability**: readiness for correct service,
- **Reliability**: continuity of correct service,
- **Safety**: absence of catastrophic consequences on the user(s) and the environment,
- **Integrity**: absence of improper system state alterations,
- **Maintainability**: ability to undergo modifications and repairs.

Availability is typically measured by the percentage of time a system is available to users (the total proportion of time a system is operational), while reliability refers to the duration of uninterrupted periods of operation (the MTBF) [@asplund2007restoring]. For example, a service that goes down for one second every fifteen minutes provides reasonable availability (99.9 %) but very low reliability.

The availability of a system monitored over a time period $[0,t]$ is expressed by dividing the total uptime by the total time of monitoring the system (i.e., the sum of uptime and downtime) [@helal2006replication]:

$$ A(t) = \frac{\sum_i u_i}{t} $$

where $u_i$ are the periods of uptime.

In case of fail-stop failures, this can also be simplified and expressed by the Mean Time To Failure (MTTF) (equation \ref{eq:mttf}) and Mean Time Between Failure (MTBF), as the MTTF describes the mean time of uninterrupted uptime:

$$ \lim_{t \to \infty} A(t) = \frac{\textrm{MTTF}}{\textrm{MTBF}} = \frac{\textrm{MTTF}}{\textrm{MTTF} + \textrm{MTTR}} $$

where in this case the MTTR is simplified also includes the MTTD. Note that this won't apply to byzantine failures, as the system is still operating even in case of a failure (which again shows that availability and reliability alone are not sufficient to describe dependability).

Different levels of availabilty and their respective downtimes are listed in the table \ref{table:availability-classes}. They are divided into availability classes based on the Availability Environment Classification (AEC) of the Harvard Research Group (HRG). Note that this notation may be outdated due to the ambiguous use of the terms disaster, reliable, available, and fault-tolerant, but is still the most complete and accepted classification scheme.

\begin{table}[h!]
    \caption[Various availability classes and the respective annual downtime]{Various availability classes and the respective annual downtime, based on the Availability Environment Classification (AEC) of the Harvard Research Group (HRG)}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}l r r X} 
        \toprule
        \thead{Class} & \thead{Availability} & \thead{Annual\\Downtime} & \thead{Description} \\
        \midrule
        Disaster Tolerant & 99.9999~\% & 31.56~s & Service must be available under all circumstances \\
        Fault Tolerant & 99.999~\% & 5.26~min & Service must be guaranteed without interruption, 24/7 service must be assured  \\
        Fault Resilient & 99.99~\% & 52.60~min & Service must be assured without any downtime within well defined time windows or at main runtime \\
        High Availability & 99.9~\% & 8.77~h & Service is only allowed to be interrupted within scheduled time windows or minimal at main runtime \\
        High Reliable & 99~\% & 3.65~d & Service can be interrupted, data integrity must be assured \\
        Conventional & - & - & Service can be interrupted, data integrity is not essential \\
        \bottomrule
    \end{tabularx}
    \label{table:availability-classes}
\end{table}

An availability of 99.999 % is referred to as _five nines_ and is the typical standard for telecommunication networks and a hit target for many site reliability engineering (SRE) departments of service providers, while popular commercial platforms provide lower levels of availability, such as Amazon Web Services (AWS), which provides 99.99% availability for their standard offering of S3 [@amazon2020s3availability]. Oftentimes, such services are geo-replicated and deployed in specific _availability zones_ so that they are available during peak operating hours for the corresponding time zones. The maintenance time (e.g. for updates and general MTTR) is then usually scheduled for the night hours. For critical business operations, a certain level of availability must be guaranteed by the service providers, which they typically specify in their _Service Level Agreements_ (SLAs). If the service provider fails to keep these availability promises, the provider declares themselves responsible to compensate for the damage incurred (the missed revenue). The confidence of having an excellent SLA that ensures high availability, among other things, can often be a decisive competitive advantage. For example, the SLA for AWS S3 guarantees customers tiered compensation based on the availability achieved in the corresponding billing cycle. As of the time of writing, partial reimbursements start when an availability of 99.9% wasn't met [@amazon2022s3sla]. The total cost of the billing cycle will be refunded if the service didn't met an availability of at least 95 % throughout this cycle. At the same time, Microsoft is offering an SLA for its Cosmos DB, a distributed NoSQL database, that guarantees partial reimbursement for certain availability regions even if a five nines availability (99.999 %) is not met, but they also cover reimbursements if latency and consistency guarantees are not met [@microsoft2022cosmossla].

When orchestrating multiple services in the context of a _service oriented architecture_ (SOA), the availability of a resulting aggregated system or business process depends heavily on all the orchestrated services as well as the orchestrating system itself, and decreases for each participating system. Ignoring network availability, overlapping unavailability periods and other factors, and assuming all services are run in a sequential order, we can roughly estimate the overall availability as

$$ A_{\textrm{serial}}(t) \approx \prod_i A(t)_i $$

where $A(t)_i$ are the availabilities of the various systems in the period of $t$, including the orchestration engine. Note that this is just a naive estimation that does not take into account failover and compensation mechanisms of the orchestration engine that can actually improve the availability again. Figure {fig:serial-availability} illustrates this for two nodes.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/sequential-service-availability.pdf}
  \caption[Availability of sequential orchestration of services]{Example for the resulting availability of a sequential orchestration of services}
  \label{fig:serial-availability}
\end{figure}

Conversely, replication is a way to drastically increase the availability of a system. By having more than one copy of important information, the service continues to be usable even when some copies are inaccessible. By providing a _High-Availability Cluster_ (HA Cluster)[^service-load-balancing] of replicas, the probability of failure of the whole cluster during a fixed period of time decreases with every replica, decreasing the estimated annual downtime. When we ignore factors like failures of the replication system itself, faults in the underlying network, and the time needed for cluster reconfigurations on node failures (as included in the cluster MTTR), we can model the availability of the cluster naively as

$$ A_{\textrm{cluster}}(t) \approx 1 - (1 - A(t))^n = 1 - U(t)^n $$

where $n$ is the number of replicas, $A(t)$ is the availability of a single node in the period of $t$, and $U(t)$ the respective unavailability. A replica configuration like this fails if all of its replicas fail. Figure {fig:parallel-availability} illustrates this for two nodes.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/replicated-node-availability.pdf}
  \caption{Example for the resulting availability of a replica set}
  \label{fig:parallel-availability}
\end{figure}

[^service-load-balancing]: Service load balancing is another technique for achieving high availability through service replication. Since this work only deals with data replication, this topic is not covered here (although some session data must be replicated between these services if they are not stateless in the first place).

As shown, the common way to achieve high availability is through the replication of data in multiple service replicas. High availability comes hand-in-hand with fault-tolerance, as services remain operational in case of failures as clients can be relayed to other working replicas. 

### Consistency {#sec:consistency}

Building scalable and reliable distributed systems requires a trade-off between consistency and availability. Consistency is a property of the distributed system that ensures that every node or replica has the same view of the data at a given point in time, regardless of which client updated the data. Ideally, a replicated data store should behave no differently than one running on a single machine, but this often comes at the cost of availability and performance. Deciding to trade some consistency for availability can often lead to dramatic improvements in scalability [@pritchett2008base].

Consistency is an ambiguous term in data systems: in the sense of the _ACID model_ (Atomic, Consistent, Isolated and Durable), it is a very different property than the one described in the _CAP theorem_ (we explain this later in this section). The ACID model describes consistency only in the context of database transactions, whereas consistency in the CAP model refers to a single request/response sequence of operations. We focus on the definition in the CAP theorem in this work. In the distributed systems literature in general, consistency is understood as a spectrum of models with different guarantees and correctness properties, as well as various constraints on performance.

A database consistency model[^consistency-origins] determines the manner and timing in which a successful write or update is reflected in a subsequent read operation of that same value. It describes the ordering guarantees for the execution of operations accessing the system and what values are allowed to be returned. There exists no one-size-fits-all approach: it is difficult to find one model that satisfies all main challenges associated with data consistency [@mkandla2021evaluation]. Consistency models describe the trade-offs between concurrency and ordering of operations, and thus between performance and correctness.

[^consistency-origins]: Originally, consistency models were discussed in the context of concurrency of operations on a single machine or set of CPUs respectively in multi-threaded environments. Discussions of consistency models in distributed systems or even distributed database systems are not substantially different, since a distributed system is basically nothing more than a collection of machines on which multiple threads and processes can run concurrently or in parallel.

There are two distinct perspectives on consistency models: the data-centric and client-centric perspective, as illustrated in figure \ref{fig:data-client-centric-perspective} [@bermbach2013towards]. The data-centric perspective analyzes consistency from a replica's point of view, where the distributed system synchronizes read and write operations of all processes to ensure correct results. Similarly, the client-centric perspective examines consistency from a client's point of view. From this perspective, it is sufficient for the system to synchronize only the data access operations of the same process, independently of the others, to ensure their consistency. This is justified because common updates are often rare and mostly access private data [@campelo2020brief]. It is sufficient to _appear_ consistent to clients, while being at least partially inconsistent between cluster nodes. Therefore, client-centric consistency models are in general weaker than data-centric models. 
<!-- In this work, we focus on the data-centric perspective because the client-centric perspective depends on it and can be derived from it. -->

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/data-client-centric-consistency.pdf}
  \caption[Data-centric and client-centric perspective on consistency models]{The data-centric and client-centric perspective on consistency models, as described by Bermbach et al.}
  \label{fig:data-client-centric-perspective}
\end{figure}

Following, a few of those consistency models are discussed, starting with the model with the strongest consistency guarantees [@steen2007distributed]. Throughout the discussion, we use the following notation (also see figure \ref{fig:consistency-timeline-notation}) to illustrate the behavior of the models by examples:

- $w(x, v)$ denotes a successful write operation of the value $v$ into the variable $x$.
- $r(x) = v$ denotes a read of $x$ that returns the value $v$.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.8\textwidth]{images/consistency-timeline-notation.pdf}
  \caption[Notation of operation invocations used throughout the work]{Notation of an operation invocation to illustrate following consistency discussions}
  \label{fig:consistency-timeline-notation}
\end{figure}

\todo{Denote everything mathematically if sufficient time and if it does not unneccessary complexity}

<!-- Strong Consistency -->

\paragraph{Strong Consistency (Linearizability).}

Strong consistency is, from the view of a client, the ideal consistency model: a read request made to any of the nodes of the replica cluster should return the same data. A replicated data store with strong consistency therefore behaves indistinguishably from one running on a single machine. A strong consistency model provides the highest degree of consistency by guaranteeing that all operations on replicated data behave as if they were performed atomically on a single copy of the data. After a successful write operation, the written object is immediately accessible from all nodes at the same time: _what you write is what you will read_[^read-after-write]. This is illustrated in \ref{fig:consistency-ordering-linearizable}. A client never sees an uncommitted or partial write. However, this guarantee comes at the cost of lower performance, reliability, and availability compared to weaker consistency models [@attiya1994sequential; @liu2013replication]: Algorithms that guarantee strong consistency properties across replicas are more prone to message delays and render the system vulnerable to network partitions, if not handled properly. Such algorithms do need special strategies to provide partition-tolerance to the cost of availability.

[^read-after-write]: For this reason, it is sometimes referred to as _read-after-write consistency_ from a client-centric view.

In the literature, strong consistency is often referred to as _linearizability_. To achieve linearizability, an absolute global time order must be maintained [@lamport1978time]. Each operation must appear to be executed immediately, exactly once, at some point between its invocation and its response [@herlihy1990linearizability]. This is illustrated in the figures \ref{fig:consistency-ordering-linearizable-overlap} and \ref{fig:consistency-ordering-linearizable-write-overlap}. If a write call returns, the client can be sure that the written data is available throughout the system, just as it would be with a single-node system. Therefore, consistent data stores are easy to use for developers because they do not need to be aware of the differences between the respective replicas of the data store. 

Strong consistency is mandatory for use cases with zero tolerance for inconsistent states and strong requirements for real-time ordering guarantees, such as

- Business process automation,
- Service orchestration (SOA),
- Financial services,
- Stock exchange trades and analysis,
- Event sourcing,
- Aviation systems.

\todo{Mention 2-phase commits (2PC)}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/strong-consistency-flow.pdf}
  \caption[Communication model for strong consistency]{Communication model between clients and replica cluster nodes to ensure strong consistency. A write is acknowledged only when it is consistent throughout all cluster nodes, therefore synchronizing the operations.}
  \label{fig:strong-consistency-flow}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/consistency-ordering-linearizable.pdf}
  \caption[Operation schedule that satisfies linearizability]{Operation schedule that satisfies the realtime ordering guarantee of the strong consistency model (linearizability)}
  \label{fig:consistency-ordering-linearizable}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/consistency-ordering-linearizable-overlap.pdf}
  \caption[A strongly consistent operation schedule with a read-write overlap]{A strongly consistent operation schedule with a read-write overlap. During the overlap, the read is allowed to return either the current or the previous write.}
  \label{fig:consistency-ordering-linearizable-overlap}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/consistency-ordering-linearizable-write-overlap.pdf}
  \caption[Linearizable operation schedule with overlapping writes]{This operation schedule is still linearizable, as the order of committing the overlapping writes is not guaranteed.}
  \label{fig:consistency-ordering-linearizable-write-overlap}
\end{figure}

\begin{table}[h!]
    \caption{Fact sheet for strong consistency}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l} 
        \toprule
        Consistency & Strongest \\
        Ordering guarantees & Real-time (for non-overlapping operations) \\
        Availability & Lowest \\
        Latency & High \\
        Throughput & Lowest \\
        Perspective & Data-centric \\
        \bottomrule
    \end{tabularx}
    \label{table:facts-strong-consistency}
\end{table}

The linearizability constraints can be further hardened, resulting in _strict consistency_. While linearizability takes overlapping operations in a relaxed way, as illustrated in figure \ref{fig:consistency-ordering-linearizable-write-overlap}, strict consistency does not give that freedom. Overlapped operations also need to be ordered in strict real time order by the time of their invocation. In practice, strict consistency is hard to implement and is rarely necessary for practical use cases, hence it is reduced to a theoretical basis only.

<!-- Sequential -->
\paragraph{Sequential Consistency.} 

<!-- This allows systems to acknowledge a write earlier than it has been accepted by a majority. ?? -->

Sequential consistency weakens the constraints of strong consistency[^linearizability-contains-sequential] by dropping the real-time property. If a write call returns, it must not neccessarily be available in a subsequent read, but when it is, then the correct order of the write operations of the originating node or process is guaranteed [@attiya1994sequential]. That is, linearizability takes care of time and sequential consistency takes care of program order only. There are two properties defining sequential consistency: first, the writes of one node must appear in the originally executed order (program order) on every node. Second, the order of writes between nodes is not specified, but all nodes must agree on a sequential order (global order) while ensuring program order. This is illustrated in \ref{fig:consistency-ordering-sequential}.

[^linearizability-contains-sequential]: It is easy to show that a schedule of operations that satisfies linearizability also satisfies sequential consistency. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/sequential-consistency-flow.pdf}
  \caption[Schematic communication model for sequential consistency]{Schematic communication model between clients and replica cluster nodes for sequential consistency. A write can be acknowledged before it is propagated consistently across all cluster nodes. For all successful writes that are committed throughout the cluster, the original order is guaranteed.}
  \label{fig:sequential-consistency-flow}
\end{figure}

All reads at all nodes will see the same order of writes to ensure sequential consistency, but not necessarily in the absolute order (by timestamps) in which clients requested the reads, as the latter can often be impractical as it can lead to reordering of operations between nodes when concurrent writes appear in the wrong order. With sequential consistency, programmers must be careful because two successive writes from different nodes (in real time) to the same value can occur in any order, resulting in unexpected overwrites.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/consistency-ordering-sequential.pdf}
  \caption[Operation schedule for the sequential consistency model]{Operation schedule that satisfies the global ordering guarantee of the sequential consistency model. The writes of $\textrm{N}_1$ and $\textrm{N}_2$ are seen by $\textrm{N}_3$ in the order of their program execution on the particular nodes, while the order of the interwoven operations does not neccessarily correspond to the real-time sequence. Both $\textrm{N}_3$ and $\textrm{N}_4$ have also agreed on one (partial) global order, while $\textrm{N}_4$ experiences updates faster.}
  \label{fig:consistency-ordering-sequential}
\end{figure}

\begin{table}[h!]
    \caption{Fact sheet for sequential consistency}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l} 
        \toprule
        Consistency & Strong \\
        Ordering guarantees & Global ordering \\
        Availability & Low \\
        Latency & High \\
        Throughput & Low \\
        Perspective & Data-centric \\
        \bottomrule
    \end{tabularx}
    \label{table:facts-sequential-consistency}
\end{table}

<!-- Causal -->
\paragraph{Causal Consistency.} 

Causal consistency relaxes the constraints of sequential consistency by removing the requirement for global order[^sequential-contains-causal]. A system is causally consistent if all operations that are _causally dependent_ must be seen in the same order on all nodes. All other (concurrent) operations can appear in any arbitrary order at the nodes, as well as the sets of causally related operations, and nodes do not need to agree on a global ordering. Causality is therefore a partial ordering on the set of all operations.

[^sequential-contains-causal]: It is easy to show that a schedule of operations that satisfies sequential consistency also satisfies causal consistency. 

An operation $b$ is causally dependent on an operation $a$ (denoted by $a \to b$) if one or more of the following conditions are met [@lamport1978time]:

\begin{enumerate}[label=(\arabic*)]
  \item $a$ and $b$ were both triggered on the same node and $a$ was chronologically prior to $b$.
  \item $a$ is a write, $b$ is a read, and $b$ reads the result of $a$.
  \item (Transitivity) $c$ is causally dependent on an operation $b$, which in turn is causally dependent on $a$: If $a \to b$ and $b \to c$ then $a \to c$.
\end{enumerate}

Two operations $a$ and $b$ are said to be concurrent if $a \not\to b$ and $b \not\to a$.

Causal consistency is mostly studied and used in geo-replication (see subsection [@sec:geo-replication]) and partial replication, as it still satisfies a latency less than the maximum wide-area roundtrip delay between replicas [@shen2015causal], and it only cares about a partial ordering that is of interest for the client or user: users are often only interested in a (causally related) subset of events, e.g., those that happened close to their location. An example to illustrate this are social media posts: when a user posts a status update and another user reads and replies to that update, there is a causal order on the two updates, and they should appear in that order to other (subscribed) users. However, when other users send totally unrelated updates, the order in that these updates appear is not important (at least from a consistency point of view). Another example are stock markets: operations on a a single stock (as a reaction to a stock value change) must be consistently ordered, while changes across different, independent stocks ca be seen in different orders.

There are some extensions to harden causal consistency slightly, namely _causal+_ (or _causal consistency with convergent conflict handling_), which leverages the existence of multiple replicas to distribute the load of read requests [@lloyd2011don], and realtime causal consistency (RTC). 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/causal-consistency-flow.pdf}
  \caption[Schematic communication model for causal consistency]{Schematic communication model between clients and replica cluster nodes for causal consistency. A write can be acknowledged before it is propagated across all cluster nodes. For all causally sequential writes that are committed throughout the cluster, the original order is guaranteed.}
  \label{fig:causal-consistency-flow}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/consistency-ordering-causal-violation.pdf}
  \caption[Operation schedule that violates causal consistency]{Operation schedule that violates causal consistency. As the writes of $\textrm{N}_1$ has already been observed by $\textrm{N}_2$ before overwriting it (assuming that its observation may have triggered the overwrite, thus they are causally related), it must be seen in this order by all subsequent reads of all nodes. Since $\textrm{N}_4$ reads the most recent (the overwriting) value before the overwritten one, it ignores the causal relation and thus violating the properties of causal consistency.}
  \label{fig:consistency-ordering-causal-violation}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/consistency-ordering-causal.pdf}
  \caption[Operation schedule that meets the requirements for causal consistency]{Operation schedule that meets the requirements for causal consistency. Since there is no causal relation between the writes of $\textrm{N}_1$ and $\textrm{N}_2$, any order of occurrence of the values in subsequent reads satisfies the properties of causal consistency.}
  \label{fig:consistency-ordering-causal}
\end{figure}

\begin{table}[h!]
    \caption{Fact sheet for causal consistency}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l} 
        \toprule
        Consistency & Moderate \\
        Ordering guarantees & Causal \\
        Availability & High \\
        Latency & Moderate \\
        Throughput & Moderate \\
        Perspective & Data-centric \\
        \bottomrule
    \end{tabularx}
    \label{table:facts-causal-consistency}
\end{table}

<!-- Eventual -->
\paragraph{Eventual Consistency.}

To achieve a higher level of consistency, synchronization between replicas is usually required, increasing the latency and even rendering the system unavailable if network connections between the replicas fail. For this reason, modern replicated systems that put emphasis on throughput and latency often forgo synchronization altogether; such systems are commonly referred to as _eventually consistent_ [@vogels2009eventually]. Eventual consistency is a weak consistency model that does not guarantee any global ordering, but only _liveness_: intermediate states are allowed to be inconsistent, but after some time, in the absence of updates, all nodes should converge, returning the same resulting state set of operations [@terry1994session]. This is illustrated in \ref{fig:consistency-ordering-eventual}. As illustrated in figure \ref{fig:eventual-consistency-flow}, in eventually consistent distributed databases, a replica performs an operation requested by a client locally without any synchronisation with other replicas and immediately acknowledges the client of the response. The operation is passed asynchronously to the other replicas and, in the case of network partitioning, can be pending for a while. The time taken by the replicas to get consistent may or may not be defined, but the model clearly requires that in the absence of updates, all replicas converge toward identical copies. This often requires _conflict resolution_ techniques.

Eventual consistency could be somehow referred to as the incarnation of the liveness property described in subsection [@#sec:safety-reliability] ("something good will eventual happen"), while safety is not neccessarily guaranteed: it depends on the use case and potential conflict resolution.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/eventual-consistency-flow.pdf}
  \caption[Schematic communication model for eventual consistency]{Schematic communication model between clients and replica cluster nodes for eventual consistency. A write is acknowledged immediately before it is propagated across all cluster nodes. The system remains available even when partitioned by allowing disconnected nodes to converge later when reconnected again.}
  \label{fig:eventual-consistency-flow}
\end{figure}

Because of the possibility of inconsistency, application developers integrating eventually consistent data stores must be explicitly aware of the replicated nature of data items in the store. Compared to strong consistency, the developer must consider the asynchronous behavior and take care that stale data does not render their application unusable, and also make their end users aware of this through clear in-application communication. We describe these considerations in detail in the subsection [@sec:consistency-decisions].

There are also use cases where eventual consistency is ideal since they do not require any ordering guarantees, for example:

- Non-threaded comments, reviews or ratings,
- Incremental counters, such as of likes in social media or views on videos.

One of the best known examples of an eventually consistent distributed system is the _Domain Name System_ (DNS). DNS is a hierarchical, highly available system that handles billions of queries every day. The values are cached and replicated across many servers, while it takes some time for the update of a particular record to propagate through DNS servers and clients.

In the database literature, eventually consistent databases are often classified as _BASE_ (Basically Available, Soft-state, Eventually consistent), as opposed to the traditional ACID requirements for _relational database management systems_ (RDBMS) [@pritchett2008base]. _Basically available_ describes that services are available as much as possible, but data may not be consistent. _Soft-state_ describes the convergence behavior, that after a certain amount of time, there is only a certain probability of knowing the actual state. In the past, NoSQL databases were often implemented to meet BASE requirements, hence being eventually consistent. This has changed over time, such as MongoDB supporting strong consistency and also ACID transactions.

Eventual consistency can become a problem when operations aren't idempotent and users repeat operations, because they don't receive on a read what they've just written. To mitigate this, another consistency model has been derived from eventual consistency, called _strong eventual consistency_. It enforces _idempotency_ and _commutativeness_ on all operations. By that, strong eventual consistency hardens the liveness property by adding a safety guarantee to the model (see subsection [@sec:safety-reliability] for reference): any nodes that have received the same (probably unordered) intermediate set of updates will be in the same state. An example of a use case for strong eventual consistency are _conflict-free replicated data types_ (CRDT), which are described in subsection [@sec:optimistic-replication]. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.75\textwidth]{images/consistency-ordering-eventual.pdf}
  \caption[Operation schedule for eventual consistency]{Operation schedule for eventual consistency. One node is partitioned from the others, returning an inconsistent, outdated state in between. When reconnected, the state of the partition converges and eventually becomes consistent again.}
  \label{fig:consistency-ordering-eventual}
\end{figure}

\begin{table}[h!]
    \caption{Fact sheet for eventual consistency}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l} 
        \toprule
        Consistency & Lowest \\
        Ordering guarantees & None \\
        Availability & High to Highest \\
        Latency & Low \\
        Throughput & Highest \\
        Perspective & Client-centric \\
        \bottomrule
    \end{tabularx}
    \label{table:facts-eventual-consistency}
\end{table}

<!-- Weak -->
\paragraph{Weak Consistency.}

As its name indicates, weak consistency offers the lowest possible ordering guarantee, since it allows data to be written across multiple nodes and always returns the version that the system first finds. This means that there is no guarantee that the system will eventually become consistent. Only writes that are explicitly synchronized are consistent. Everything else in between is unordered, but at least the same set of operations on all nodes. This requires programmers to explicitly synchronize operations. Synchronized operations are sequentially consistent, as they are seen by all processes in the same order. Weak consistency with explicit synchronization is uncommon in distributed systems, but a common model in concurrent, multi-threaded programming.

\begin{table}[h!]
    \caption{Fact sheet for weak consistency}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X} 
        \toprule
        Consistency & Lowest to None \\
        Ordering guarantees & Only explicitly synchronized operations (sequential), else none \\
        Availability & High \\
        Latency & Low to Lowest \\
        Throughput & Highest \\
        Perspective & Data-centric \\
        \bottomrule
    \end{tabularx}
    \label{table:facts-weak-consistency}
\end{table}

So far, this subsection has briefly introduced consistency models[^more-consistency-models]. Table \ref{table:consistency-models} summarizes all the discussed consistency models. The next subsection explains how tradeoffs are made between consistency, availability, and latency, and how designers of distributed database systems can choose a consistency model (or multiple models) that meets the needs of their applications.

\begin{table}[h!]
    \caption[Consistency models in descending order of strictness]{Consistency models in descending order of strictness and at the same time in ascending order of availability}
    \vspace{1ex}
    \centering
    $
    \left\uparrow
        \rotatebox[origin=c]{90}{Strictness \& Ordering Guarantees}
        \hspace{2ex}
      \def\arraystretch{1.5}
        \begin{tabularx}{0.85\textwidth}{>{\bfseries}l X} 
            \toprule
            \multicolumn{2}{c}{\thead{Data-centric}} \\
            \midrule
            Strong consistency & After a successful write operation, the value can be read at all nodes immediately \\
            Sequential consistency & No immediate read, but writes have to be seen in the same order at every node  \\
            Causal consistency & Only causally related writes must be seen in the same order at every node \\
            Weak consistency & Only writes that are explicitly synchronized are consistent. Everything else in between is unordered \\
            \midrule
            \multicolumn{2}{c}{\thead{Client-centric}} \\
            \midrule
            Eventual consistency & After the last update, all replicas all replicas converge toward identical states, eventually becoming consistent, but with no time and ordering guarantees \\
            \bottomrule
        \end{tabularx}
        \hspace{2ex}
    \right\downarrow
    \rotatebox[origin=c]{-90}{Availability \& Throughput}
    $
    \label{table:consistency-models}
\end{table}

[^more-consistency-models]: In addition to the models discussed in this work, there are other consistency models. Since they do not play an important role for the kind of systems discussed in this work, we will not investigate them further, but provide a uncommented list of some of them: session consistency (e.g. monotonic read, monotonic write, read-my-write, write follows read), bounded staleness, consistent prefix and more.

#### The CAP Theorem

Why can't we always have strong consistency? The CAP theorem (CAP stands for Consistency, Availability and Partition Tolerance), also known as Brewer's Theorem, named after its original author Eric Brewer [@brewer2000towards], asserts that the requirements for strong consistency and high availability cannot be met at the same time and serves as a simplifying explanatory model for consistency decisions that depend on the requirements for the availability of the distributed system in question [@gilbert2002brewer].

\begin{figure}[h]
  \centering
  \includegraphics[width=0.4\textwidth]{images/CAP.pdf}
  \caption{Illustration of the CAP theorem}
  \label{fig:cap}
\end{figure}

In this chapter, the three properties of the CAP theorem have already been presented. We summarize them here in the context of this theorem:

- **(Strong) Consistency**: The system satisfies linearizability, thus all clients see the same data at the same time.
- **Availability**: The system operates even in case of node failures (is fault-tolerant), so read and write requests always receive a response.
- **Partition Tolerance**: The system continues to work even under arbitrary network partitioning. This is a mandatory requirement in distributed systems.

The theorem states that only two of those properties can apply at the same time. This results in the following classes:

- **CA**: A network problem might render the system unavailable. Not suitable for any distributed system. Traditional RDBMS fall under this class.
- **CP**: Strong consistency is guarenteed even in the case of network partitions, but some data may become unavailable.
- **AP**: The system is available even in the case of network partitions, but potentially returning inconsistent data.

This classes reduce the trade-off between availability and consistency to strict binary terms, when in fact, as the various consistency models show, the trade-off is gradual in nature. 

In general, partition tolerance should always be guaranteed in distributed systems (as described in section [@sec:possible-faults]). Therefore, the trade-off is to be made between consistency and availability in the case of a network partition. Eric Brewer revisited his thoughts on the theorem, stating that trade-offs are possible for all three dimensions of the theorem: by explicitly handling partitions, both consistency and availability can be optimized [@brewer2012cap]. This gives credit to the gradual nature of the trade-off in consistency models.

The theorem is often interpreted as a proof that eventually consistent databases have better availability properties than strongly consistent databases. There is sound critic that in practice this is only true under certain circumstances: the reasoning for consistency trade-offs in practical systems should be made more carefully and strong consistency can be reached in more real-world applications then the CAP theorem would allow [@kleppmann2015critique]. In the next two subsections, we present two critics or extensions to the CAP theorem.

#### The PACELC Theorem

The author of the PACELC theorem criticizes the CAP theorem to just focus on failures [@abadi2012consistency]. They described the PACELC theorem to extend the CAP theorem by another trade-off in the case of normally operating systems with no partitioning failure: In the case of network partitioning (P), the CAP rule still applies, where one must choose between availability (A) and consistency (C). But else (E), when there are no network partitions and the system is operating normally, one still has to choose between latency (L) and consistency (C).

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/pacelc.pdf}
  \caption[Illustration of the PACELC theorem]{Illustration of the PACELC theorem, showcasing the trade-offs in the case of partitioning and also in the absence of network partitioning}
  \label{fig:pacelc}
\end{figure}

The author introduced this theorem to support decision making in the design and implementation of a distributed database system, since consistency decisions must also be made outside of network partitions: the trade-off between consistency and latency is often even more important and arises as soon as a distributed database system introduces replication.

\todo{Further explanation if sufficient time}

#### The CALM Theorem {#sec:calm}

The CALM theorem (Consistency As Logical Monotonicity) arose from a critique of the CAP theorem. It helps to understand whether a distributed problem can be solved with a strong or weak consistency model. While a strong consistency model always requires some form of coordination between nodes which increases latency, a weak model such as eventual consistency can eschew coordination entirely, but often at the cost of violating correctness properties. The CALM theorem allows the system designer to decide whether a weak consistency model can be applied without compromising correctness by considering possible monotonic properties. The theorem says "a problem has a consistent, coordination-free distributed implementation if and only if it is monotonic [@hellerstein2020keeping]".

Monotonicity is defined as follows: A problem $P$ is monotonic if for any input sets $S$, $T$ where $S \subseteq T$, $P(S) \subseteq P(T)$. Figure \ref{fig:monotonic-function} illustrates this using a simple function example.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/monotonic-function.pdf}
  \caption{A curve for a nondecreasing monotonic function}
  \label{fig:monotonic-function}
\end{figure}

For a distributed problem to be monotonic, all operations must be designed in a way that they are 

\begin{enumerate}[label=(\arabic*)]
  \item Associative: $a \circ (b \circ c) = (a \circ b) \circ c$,
  \item Commutative: $a \circ b = b \circ a$,
  \item Idempotent: $a \circ a = a$.
\end{enumerate}

The challenges of the theorem are

- to determine whether a problem can be described by a monotonic specification at all,
- and, if this is the case, to design and practically implement this specification.

One way to achieve this behavior is to apply the _derived monotonic state pattern_ [@braun2022calm]. The pattern is illustrated in figure \ref{fig:derived-monotonic-state-aggregate}, as initially described by Braun. The pattern can be applied by factoring out every operation on the state object into _immutable aggregates_, such as _domain events_. We will illustrate this with the example of a shopping cart. Domain events in a shopping cart are the insert and removal of items, denoted as `itemInserted` and `itemRemoved`. In a naive implementation, both events could be modeled as operations on a mutable state (the shopping cart entity). But with a weak consistency model, the order of these operations can not be guaranteed, as illustrated in figure \ref{@fig:shopping-cart-naive}.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/shopping-cart-naive.pdf}
  \caption[Naive implementation of a shopping cart]{Naive implementation of a shopping cart with weak consistency. $N_2$ received the operations in a different order than $N_1$, rendering a wrong final state.}
  \label{fig:shopping-cart-naive}
\end{figure}

But this problem can be implemented in a monotonic fashion: instead of applying the domain events on a single mutable state (an _activity aggregate_), they could just be persisted as immutable aggregates in an append-only manner, resulting in two separate monotonically growing sets. Once the final state needs to be observed, it can be derived from the domain events by putting them together on demand[^state-management-libraries]. The resulting state then describes a derived aggregate, expressed by the count of `itemInserted` minus the count of `itemRemoved` for each particular item.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/shopping-cart-monotonic-aggregate.pdf}
  \caption[Monotonic implementation of a shopping cart]{Monotonic implementation of the shopping cart problem. Both nodes will derive the same final state without the need for coordination.}
  \label{fig:shopping-cart-monotonic-aggregate}
\end{figure}

Unfortunately, this monotonic behavior is not applicable to all types of operations. There are events that are naturally causally dependent on previous events. In our shopping cart example, this could be a checkout[^event-sourcing]: neither the add nor delete operation commutes with
a final checkout operation. If a `checkout` operation message arrives at a node before some insertions of the `itemInserted` or `itemRemoved` events, those events will be lost.

[^state-management-libraries]: This is also how modern state management systems like MobX or Redux (that are heavily used in frontend development) do this to ease complex implementation of concurrency problems on single devices by applying _functional reactive programming_ paradigms.

[^event-sourcing]: In _event sourcing_, monotonic characteristics can be useful. However, to be able to do _time travel queries_, strong consistency and realtime properties are needed again to derive any intermediate state, at least for all causally related events.

The monotonic property of a problem means that the order of operations does not matter at all. Consequently, in the case of network partitions, both consistency and availability are possible in a monotonic problem, since replicas will always converge to an identical state on all nodes when the partition heals, and this without the need for any conflict resolution or coordination mechanisms.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/derived-monotonic-state-aggregate.pdf}
  \caption[Activity aggregate factored out into immutable and derived aggregates]{Factoring out a nontrivial activity aggregate into immutable and derived aggregates to acquire a monotonic state aggregate allows for a weaker consistency model and therefore lower latency without actually compromising consistency, according to the CALM theorem.}
  \label{fig:derived-monotonic-state-aggregate}
\end{figure}

### Partitioning and Sharding {#sec:partitioning}

<!--
https://dimosr.github.io/partitioning-and-replication/ 
https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html
-->

While replication increases the dependability of a distributed system and also helps to reduce latency in geo-replicated systems (described in more detail in subsection [@sec:geo-replication]), _partitioning_ is neccessary to scale out, i.e. to distribute the workload across multiple nodes. Partitioning is the method of breaking a large dataset into smaller subsets. For distributed systems serving many different clients or users, partitioning is essential to maintain both the performance and availability of the system at a high level. With partitioning, the number of nodes of a system grows with the number of clients. As described in subsection [@sec:scalability], this happens in general in a linear fashion.

Partitioning helps to improve the performance of both writes and reads: in partitioned databases, when queries only access a fraction of the data that resides in a subset of the partitions, they can run faster because there is less data to scan. This reduces the overall response time to read and load data. Data partitions can also be stored in separate file systems or hardware with different characteristics, depending on the read or write requirements for the content. Thus, data that is accessed very frequently can be held in in-memory caches or fast SSDs, while on the other hand, very infrequent accesses can be stored on dedicated archived mass storage.

There are two ways to partition data: Vertical and horizontal partitioning. Both techniques can be combined. They are described in the following paragraphs and their differences are illustrated in figure \ref{fig:partitioning}.

<!-- Two approaches of partitioning -->

<!-- TODO first introduce vertical partitioning, then horizontal. Then introduce sharding as horizontal partitioning distributed across multiple machines. in the literature we speak of _sharding_ when the data is distributed over several machines -->

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/partitioning.pdf}
  \caption[Vertical vs horizontal partitioning]{a) Vertical vs b) horizontal partitioning, illustrated using a relational database table. In a), the table is split by attributes. In b), the table is split by records, using a lexicographical grouping.}
  \label{fig:partitioning}
\end{figure}

\paragraph{Vertical Partitioning.}

In vertical partitioning, data collections[^data-collection] such as tables are partitioned by attributes. A collection is partitioned into multiple collections, each containing a subset of the attributes. An example of this are database systems where large data blobs that are rarely queried are stored separately (i.e., only when accessing the full set of details for a single record, but not when listing multiple records). Database normalization is also an example of vertical partitioning. Vertical partitioning comes in particularly handy when the partitioned data can be stored in separate file systems or hardware with different characteristics, depending on the read or write frequency and requirements of the different record contents. These requirements may also include dependability and consistency, so that, for example, less critical data may be given an eventual consistency model and lower availability guarantees. On the downside, vertical partitioning can make querying more complicated, so partitioning decisions must be made with proper justification and based on usage estimates.

The need for vertical partitioning can also be reduced by better upfront data design, such as splitting data in trivial facts and deriving an aggregated state instead of managing the a state by continously updating a single data record (cf. subsection [@sec:calm]). 

[^data-collection]: We will use the term _data collection_ to describe structured data as well as semi-structured data that belongs to a certain schema (that describes the structure of the data collection) in any kind of data stores: tables in relational databases, streams in event stores, topics in message brokers, or buckets in file storage systems, to mention a few.

In general, vertically partitioned data is not distributed across multiple nodes, as this would slow down queries: The chances that partitioned attributes of a single dataset will be requested in a single query are high. When partitions are distributed, partitioning strategies should be defined so that cross-partition queries are rare. For this reason, we will not discuss this issue further in this work.

\paragraph{Horizontal Partitioning.}

In horizontal partitioning, data collections are partitioned by records, while all partitions of the same collection share the same schema. The collection is split by one or more grouping criteria, such as a hash, a certain attribute value or by ranges of attribute values (e.g., by quantitative attributes, date periods, groups of customers or lexicographic ordering). Which split criteria to choose heavily depends on both the use case and technical considerations. With horizontal partitioning, the workload can be distributed across multiple indexes on the same logical server. By choosing well-justified partitioning criteria, frequently visited indexes can be kept small to increase transaction and query throughput.

\paragraph{Sharding.}

Sharding goes beyond partitioning tables on a single machine by distributing horizontal partitions across multiple machines [bagui2015database]. This allows to distribute the transaction and query load across multiple servers (both logical or physical). 

The optimal _shard key_ (resulting from the split criteria in the context of sharding) allows for _load balancing_, i.e. it distributes the workload as evenly as possible across the shards. It also maximizes coherence and locality within a partition and reduces the number of  queries and transactions across partitions. For instance, the MongoDB authors recommend choosing shard keys with a high level of randomness for write scaling and high locality for range queries [@kookarinrat2015analysis]. Good partitioning moves computation close to the node where that data resides, and the master replica of a shard close to where the most clients will write to it (in case of replication with a strong leader, see subsection [@sec:replication-protocols]). Such sharding strategies can also support geographical scalability, as shown in subsection [@sec:geo-replication]. It also takes the access frequency and importance of every partition into account, so that database users can specify different strategies for security, access permissions, management, monitoring, and backups of the individual partitions. 

To find such a shard key, the distribution of operations over the data collection should be considered. That is, the split should not result in a heavy workload on some partitions while other partitions have a modest workload. A common approach to approximate this—assuming that the estimated workload is evenly distributed—is to split so that each shard contains a similar amount of data. For example, using the first letter of a customer's name results in an uneven distribution because some letters are more common than others. Instead, using a hash of the record ID helps distributing the data more evenly across the partitions.

When querying or writing to the data store, the shard keys for each record to be written and each cursor resulting from the query are calculated and used to look up the corresponding partitions.

Different horizontal partitions can also be served with different dependability and consistency requirements: e.g., data from customers who pay for a better SLA can be provided with higher availability guarantees than data from freemium users.

\paragraph{Rebalancing.}

Over time, the content and form of a data collection may change, and the distribution of data in the shards may deviate from the distribution originally assumed, especially if _scheme evolution_ occurs. In addition to that, the network of the sharded application itself could be reconfigured, for example, by adding data centers or upscaling machines, or in response to a failure or even as a temporary mitigation of a disaster that renders an entire data center unavailable. When this happens, the load on the shards is no longer balanced, resulting in overloaded shards that become slower, which in turn leads to an overall increase in latency and lower availability. To mitigate this, a sharded system must be continuously monitored and _rebalanced_ in response to such an event or even proactively when a specific trend is apparent. Rebalancing should ideally be done in such a way that users do not notice it, meaning it should not slow down the system or even reduce its availability [@weingartenzero]: rebalancing should not be part of the MTTR (Mean Time To Repair; for reference, see subsection [@sec:availability]). Avoiding shard keys closely coupled to actual record values and instead using consistent hashing algorithms to generate sharding keys generally allow for easier rebalancing [@lamping2014fast].

\todo{TODO consistent hashing in Cassandra if sufficient time}

\paragraph{Partitioning and Replication.}

Neither replication or partitioning alone can make a system truly scalable and available. But if both techniques are applied, distributed databases can scale with their users, the data, and the operations executed on it, they can provide world-wide high availability, and they can even withstand data center outages. In common replicated and partioned architectures, the system is divided into _availability groups_. An availability group consists of a set of user databases that fail over together. It includes a single set of primary databases and multiple sets of replicas. Such groups are often deployed in multiple data centers in different geographic _availability zones_ to provide additional disaster resilience. Figure \ref{fig:load-balanced-sharding} illustrates this architecture. 

Load balancing becomes more challenging because replicas must be considered when distributing the load among the nodes. In large, horizontally scaled distributed systems, the _replication factor_ (the number of replicas per shard) is several magnitudes smaller than the number of total nodes. The load balancer (if it is a centralized agent) or the load balancing protocol (if the load balancer itself is decentralized) must maintain a distribution of replicas that balances the load on the nodes, which means that replicas can be moved between nodes during rebalancing.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/load-balanced-sharding.pdf}
  \caption[Representation of a common replication and partitioning scheme]{Simplified representation of a common replication and partitioning scheme for high availability (with a replication factor of 3). Shards are placed in availability groups (data centers) that are close to their most frequent writers. They are replicated across availability groups so that they can be read with low latency and remain available even in the event of a data center outage.}
  \label{fig:load-balanced-sharding}
\end{figure}

<!-- now describe the difficulties and costs -->
\paragraph{Transactions on Partitioned and Replicated Databases.}

Not only the replicated data within a partition, but also the transactions across partitions must follow a consistency model that meets the requirements of the system. In addition to replication between replicas of a shard, transactions must also be ordered and the atomicity of each transaction must be guaranteed. In general, transactions should be linearizable. Following the atomicity property of the ACID schema, every transaction should be applied to all shards it affects, or none at all.

Providing consistency and atomicity in the execution of transactions in a partitioned and replicated database system is a substantially greater challenge than simply arranging operations in a single replica group, since servers in different shards do not see the same set of operations but must still ensure that they execute cross-shard transactions in a consistent order.

Existing systems generally achieve this using a multi-layer
approach, as illustrated in figure \ref{fig:standard-partitioned-architecture}. A replication protocol is used to provide fault-tolerance and dependability inside of a replica group of a shard. Across shards, an commitment protocol provides atomicity, such as the _two-phase commit_ (2PC[^2PC]), which is combined with a concurrency control protocol for isolation, e.g. two-phase locking [@li2017eris]. With geo-replication, even another layer can be added on top to mirror the whole system into multiple geographical regions. Each of this layers adds its own coordination overhead, increasing the number of network round trips significantly and therefore the resulting latency, as illustrated in figure \ref{fig:2pc-plus-consensus}. NoSQL database systems therefore often forgo transactions altogether to improve availability and latency (see BASE properties of eventual consistent systems in subsection [@sec:consistency]), while NewSQL database systems again allow for transactions and provide ACID properties—mitigating the coordination problem through coordination-free replication and transactions across shards, as shown in subsection [@sec:coordination-free-replication].

[^2PC]: The two-phase commit protocol ensures that all participants in a transaction agreed to run all the operations in the transaction, or none. In the first phase of the protocol (the voting phase), the approval or rejection of the commitment of the changes from all participants is collected. If all participants agree, they will be notified of the result (the commit phase) and all partial transactions will be executed (and ressource locks lifted); otherwhise, the transaction will be rolled back. Thus, since all members must be reachable for a transaction to work, 2PC is also referred to as an "anti-availability" protocol [@helland2016standing].

\begin{figure}[h]
  \centering
  \includegraphics[width=0.85\textwidth]{images/standard-partitioned-architecture.pdf}
  \caption{Common architecture for a partitioned and replicated data store}
  \label{fig:standard-partitioned-architecture}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.85\textwidth]{images/2pc-plus-consensus.pdf}
  \caption[Coordinating transactions with 2PC and replication]{Coordination between shards and replicas for consistent transactions with traditional two-phase commits (2PC) and replication}
  \label{fig:2pc-plus-consensus}
\end{figure}

### Geo-Replication {#sec:geo-replication}

The replication techniques discussed so far in this work describe the replication of data within single clusters and data centers to improve the dependability and fault-tolerance of the system. Additionally, to reduce the latency when accessing the system and data from different geographical regions and to hedge against geographically limited disasters, data can be further replicated across clusters in different geographical regions, which is referred to as geo-replication or _cross-cluster replication_. There are several strategies for geo-replication, such as using different replication protocols on multiple layers for intra and inter-cluster replication including asynchronous _data mirroring_ techniques or just extending the intra-cluster replication protocol across clusters.

\paragraph{Disaster Resilience.}

Data center failure is one of the most threatening events, as it results in all systems in the data center becoming unavailable; in the worst case, there is serious data loss. Severe outages are not rare—in a recent 2021 survey on data center outages, 6% of respondents reported severe outages in the past year [@uptime2021outage].

Many cloud vendors therefore recommend to run your data stores and services on at least two different _availability zones_. This eliminates the risk of a single data center to be a single point of failure, and the risk of unavailability and locally limited faults is significantly reduced.

In a globally distributed database system, there is a direct correlation between the level of consistency and the durability of data in the event of a region-wide outage.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/availability-zones.pdf}
  \caption[Geo-replication scheme common among cloud infrastructure providers]{Geo-replication scheme common among cloud infrastructure providers: to provide disaster resilience, data is replicated to multiple availability zones in a single region. In addition, they can be mirrored to secondary regions to provide data locality for lower read latency, as well as resilience in the event of large-scale disasters.}
  \label{fig:availability-zones}
\end{figure}

\paragraph{Reduced Read Latency.}

When it comes to read latency, it is recommended to place read replicas in close geographic proximity to clients to ensure _data locality_. This reduces the round-trip time for requests and thus the read latency for clients. Figure \ref{fig:data-locality} illustrates this.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/data-locality.pdf}
  \caption[Mirroring data into read-only replicas in different regions]{By mirroring data into read-only replicas in different regions, data locality is increased and read latency can be reduced}
  \label{fig:data-locality}
\end{figure}

\paragraph{Increased Write Latency.}

\todo{Reference this later in evaluation and also in cost of replication}

In the strong consistency model, extending the intra-cluster replication protocol across clusters can be very expensive, as more nodes must take part in coordination to reach consensus. Replicating across wide-area networks adds a minimum round-trip time to every request, which can be tens of milliseconds or more across continents. The lower bound of possible latency optimization here is defined by the speed of light: for example, the minimum roundtrip to coordinate replicas on AWS data centers across the Atlantic from Dublin, Ireland to North Virginia, USA ($\char`\~11.000$ km round-trip distance) is $\char`\~36.7$ ms (at the time of this writing, the actual round-trip ping time was $70.69$ ms). While read latency from geo-replicas near the location of the client is reduced, the write latency increases substantially. With synchronous writes, this reduces the maximum throughput to $27$ writes per second.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/geo-replication-roundtrip.pdf}
  \caption[Round-trip for coordinating between two replicas across continents]{Illustration of the round-trip for coordinating between two replicas across continents}
  \label{fig:geo-replication-roundtrip}
\end{figure}

For geo-replication, a client-centric consistency model is oftentimes sufficient, as not all data must be available for every client, leading to partial replication. A causal consistency model with partial replication can also be sufficient in this case, as it will only care about causaly related data, but it is difficult to implement [@hsu2018causal].

To prepare for disaster recovery, a trade-off between consistency and latency in the non-disaster case needs to be made, and is made on economical risk calculations: what's the cost if stale data is lost forever in the rare case of a disaster? When organizations develop a _business continuity plan_, they need to know the maximum number of lost writes the application can tolerate when recovering from a disaster. The time period of writes that an organization can afford to lose before significant damage occurs is called the Recovery Point Objective (RPO). An RPO of 0 means that strong consistency is required across geo-replicas. It should be noted that a small fraction of the system's data will always require strong consistency at a global level, mainly metadata required to manage and monitor the overall state of the system.

\paragraph{Mirroring.}

The least complex approach, which also ensures that the ordering of operations is preserved, is mirroring of a primary replica into read-only secondary replicas[^geo-replica]. This can be done with less coordination between the nodes, while it does not guarantee that every write can be read on the secondary replicas immediately: it does not provide strong consistency. During runtime, the secondary replica is updated asynchronously as a _hot backup_, so writes on the primary replica are not blocked. This can happen in a push manner, so that the primary replica send new, committed operations to the secondary replica, or in a pull manner, so that the secondary replicas fetch the latest operations continously from the primary.

[^geo-replica]: Note that the term "replica" here describes a full cluster or even a full availability zone, not just a single intra-cluster node.

In state machine replication (which is discussed in subsection [@sec:state-machine-replication]), this can also be done via snapshot installations.

Mirroring is also often used to clone data from a production cluster into a development or testing environment. Extending the distributed architecture to an edge-cloud network, these approaches are also suitable for feeding edge cluster data into one central cloud cluster (which could also be geo-replicated).

### Edge Computing

Edge computing (sometimes refered to as _fog computing_) is a paradigm that emerged to cope with the issue that traditional cloud computing is oftentimes no longer sufficient to support the high volume of data processing. In edge computing, operations are executed at the edge of the network, meaning there is an additional layer between the client's device and the cloud on the peripherals of the network that is very close to the client and provides data-locality. It is the consequent next step of geo-distribution. At the edge of the network, services are in general lightweight for local, small-scale data storage and processing [@cao2020overview].

As 5G networks become more widespread, new opportunities for the development of high-volume data processing applications arise, which are naturally suited to edge computing: 5G has the advantages of small delay, large bandwidth and large capacity, which solves many problems encountered in the traditional communication field, but also leads to the rapid growth of data volume. It allows for devices on all layers to share massive volumes of data. Therefore, the development of edge computing technology is closely related to 5G[@hassan2019edge].

An edge cloud network comprises the following three layers (as illustrated in figure \ref{fig:edge-cep-architecture} using an event processing example):

- **Terminal Layer**: The terminal layer, also referred to as the _sensor layer_, consists of all types of devices connected to the edge network, including mobile and Internet of Things (IoT) devices (such as sensors, smartphones, smart cars, or cameras). In this layer, the device is not only a data consumer, but also a data provider. Consequently, millions of devices in this layer collect raw data and upload it to the edge layer above, where it is stored and computations are performed.

- **Edge Layer**: The edge layer is the heart of the edge computing architecture. It is located at the periphery of the network and provides computational power and storage for large volumes of data in close proximity to clients. It stores and computes the data uploaded by the end devices and sends the computation results (usually aggregates of the processed data) to the cloud layer above.

- **Cloud Layer**: The cloud layer integrates all the pre-processed data from the devices on the edge layer. It permanently stores the data and creates global aggregates of all the sub-aggregates that where derived in the edge layer. If the cloud layer should persist all raw data from the terminal layer or just aggregates is up to the requirements of the application provider. The cloud layer is also responsible for computational-intense tasks, such as machine learning applications on data streams fed in from the edge layer.

\paragraph{Consistency Decisions on the Edge.}

Providing high consistency levels for high-volume data processing applications slows down the whole processing. Even without replication, the round-trip time caused by the physical distance between clients and servers in centralized systems can add significant delay to the response of a request. By physically moving the computation closer to the origin of the data, latency can be significantly reduced. Furthermore, edge-computing allows for a multi-layered consistency model design: the edge appliance can process writes in close proximity to the user with lower network round-trip times and feed this data to the central cloud system, which can be geo-replicated for disaster resilience. This makes the entire system likely to be causally consistent, but at least eventually consistent: writes to the edge appliance can be executed with strong consistency, so that recent successful writes of interest to the current client are immediately readable. Across the edge network, however, it depends on how the data is synchronized with the cloud. While clients near the edge nodes they wrote to will see the writes immediately, all other clients will see them later. In practice, this may not be a problem. It all depends on the design of the application on the different layers: ideally, only derived monotonic aggregates are allowed on the cloud layers, while all non-monotonic operations take place on the edge (see the CALM theorem in subsection [@sec:calm]). In the current literature, there are approaches to apply weaker consistency models to services that previously offered strong consistency, to meet edge-computing requirements [@jeffery2021rearchitecting].

\paragraph{Stream Processing and Complex Event Processing on the Edge.}

Edge computing naturally comes up when talking about _stream processing_ (SP) and _complex event processing_ (CEP) [@buchmann2009complex]. Complex event processing applications are moving to the edge to be able to cope with the increasing volume and throughput of events created by sensors and mobile devices[@dhillon2018edge] [@mondragon2021experimental]. The Internet of Things and 5G networks have raised the bar for the capabilities of the underlying event processing architecture, especially for real-time applications. In big data processing tasks, the data records processed are often domain facts and therefore events. Append-only data structures are optimized to work in these environments.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/edge-cep-architecture.pdf}
  \caption[Complex event processing in edge-cloud systems]{Complex event processing in edge-cloud systems}
  \label{fig:edge-cep-architecture}
\end{figure}

In figure \ref{fig:edge-cep-architecture}, the edge appliance derives aggregates (i.e., complex events), and only those aggregates are synchronized with the cloud, where they are further processed and their results can be read by clients again. This provides strong consistency across all layers. This significantly reduces latency, and available computing resources are being used more efficiently compared to a cloud-only deployment of the same application.

### Cost of Replication {#sec:cost-of-replication}

\epigraph{The first principle of successful scalability is to batter the consistency mechanisms down to a minimum, move them off the critical path, hide them in a rarely visited corner of the system, and then make it as hard as possible for application developers to get permission to use them.}{--- \textup{James Hamilton, VP \& Distinguished Engineer at Amazon}}

Now that we have discussed the benefits of replication for fault-tolerance, reducing read latency in edge computing and geographically distributed networks, and increasing dependability in general, we need to discuss the downsides: the _cost of replication_. Depending on the degree of fault-tolerance, consistency decisions, and the replication protocol chosen, performance can drop dramatically, especially for write operations. In the following paragraphs, we explain the different dimensions of replication costs, and then briefly discuss appropriate strategies for cost-benefit trade-offs in the following subsection.

\paragraph{Increased Latency and Lower Throughput.}

A modification on one replica triggers the modification on the other replicas, which must be coordinated depending on the expected consistency level. This messaging and coordination overhead degrades the overall write performance of the system. As shown in the discussion of consistency models in subsection [@sec:consistency] and described by the PACELC theorem, write latency increases with higher consistency levels. With increasing size of the cluster, the latency is expected to increase at the same time. And as we have shown in subsection [@sec:geo-replication], when we replicate across geographic regions with high levels of consistency, we expect even higher latency because we have to cope with the limits of the speed of light.

\paragraph{Availability Trade-Off.}

As the CAP theorem states, there is a trade-off between consistency and availability. To remain consistent, we must avoid split-brain situations, which reduces overall availability. But in general, availability is remarkably higher compared to running database systems in single-node mode, as we showed in [@sec:availability], and in practice even systems with strong consistency can provide five-nine times availability, as we have demonstrated anecdotally.

\paragraph{Hardware Costs.}

As we have shown in subsection [@sec:possible-faults], a cluster needs at least $2k + 1$ nodes to reach consensus while tolerating up to $k$ fail-stop failures, while to adress byzantine failures, at least $3k + 1$ nodes are neccessary. That is, to make a system fault-tolerant, additional hardware of the same class is required, multiplying the cost of acquiring, operating, and maintaining that hardware.

\paragraph{Application Complexity.}

We have shown that with strong consistency, application developers can expect the distributed database system to behave exactly like a single-node system (see subsection [@sec:consistency]). However, with weaker consistency models, developers must expect to receive stale data and the database system API may look different to provide the hooks and callbacks needed to deal with the asynchronous behavior of such systems.

For developers who implement a replication protocol in their database system, the implementation complexity increases manifold. To be able to coordinate between replicas, all read and write operations must now be designed to be functional, side-effect and context-free, serializable, and atomic, so they can be sent and executed across nodes via messaging. As the complexity of the implementation increases, so does the possibility of causing bugs through faulty code. They must verify that their implementation meets the expected consistency and dependability requirements. Fortunetally, there is the $\textrm{TLA}^{+}$ (temporal logic of actions) specification language for modeling and verification of such distributed programs. The $\textrm{TLA}^{+}$ language was introduced by Leslie Lamport in 1999 and comes nowadays with a model checker and a proof system [@lamport1999specifying]. As an anecdotal example of this, model checking with $\textrm{TLA}^{+}$ uncovered bugs in the DynamoDB database service on AWS that might never have been uncovered by interactive debugging, as some of these bugs required numerous deeply nested steps of state traces [@newcombe2014aws].

#### Deciding for Consistency {#sec:consistency-decisions}

In order to decide on a consistency model, several factors must be taken into account. The actual consistency model to decide for depends on the use case of the distributed database system. Therefore, many popular DDBS allow the developers to select the consistency model of their choice by configuration. When deciding for a consistency model, it makes sense to base the decision on the dependability properties that are neccessary for the given use case (cf. the bank transfer example in subsection [@sec:safety-reliability]).

There are even attempts to formally describe the consistency model needs at the level of operations: Gotsman et al. propose a proof rule for determining the consistency guarantees for different operations on a replicated database that are sufficient to meet given data integrity invariants [@gotsman2016cause].

\paragraph{Challenges of Weaker Consistency.}

When applying a weaker consistency model, especially eventual consistency (cf. section [@sec:optimistic-replication]), challenges arise on the consuming application side. While a strongly consistent distributed system is similar to a single-node system for the consumer and makes it easy for the developer to use because its API is clear, stale data and long-running asynchronous behavior must be handled appropriately when talking to an eventual consistent system, which makes it a completely different API. Consequently, eventual consistency is not suitable for all use cases. 

\paragraph{Multiple Levels of Consistency.}

The weaker consistency models generally violate crucial correctness properties compared to strong consistency. A compromise is to allow multiple levels of consistency to coexist in the database system, which can be achieved depending on the use case and constraints of the system. It is even possible to provide different consistency models per operation or class of data. An example is the combination of strong and causal consistency for applications with geographically replicated data in distributed data centers [@bravo2021unistore]. In general, a microservice-oriented architecture that wraps up multiple services into a single application is strongly advised, as each microservice can provide its own specific and well-reasoned consistency model suitable for its particular purpose.

\paragraph{Constantly Review Decisions.}

When stronger consistency models increase the latency of a system in an unacceptable way and there are no other ways to mitigate this, eventual consistency may be considered. It can dramatically increase the performance of a system, but it must fit the use cases of the system and its applications, and it means additional work for developers. At least, it is worth questioning again and again whether strong consistency is really mandatory: even popular systems like Kubernetes undergo this process, as current research seeks to apply eventual consistency to meet edge-computing requirements [@jeffery2021rearchitecting]. As the authors of the CALM theorem describe, instead of micro-optimizing strong consistency protocols, it is often more effective to question the overall solution design (i.e., perhaps a monotonic solution can be found) and minimize the use of such protocols (cf. subsection [@sec:calm]). At the same time, systems that have originally been eventually consistent could also benefit from re-architecting into stronger consistency to be easier to use and understand for both users and developers.

There are examples that anecdotally show that high availability and high throughput are possible even under strong consistency constraints: AWS S3 introduced strong consistency for their file operations[^s3-eventual] recently in 2021 [@amazon2021s3consistency]. They even claim to deliver strong consistency "without changes to performance or availability", compared to the earlier eventual consistency.

[^s3-eventual]: Bucket configuration operations are still eventual consistent in AWS S3 at the time of this writing.

In essence, it is worthwhile to constantly challenge applied consistency models as use cases change or new technologies and opportunities emerge.

\paragraph{Let the User Decide.}

When deciding on a consistency model for a distributed database system, it is important to recognize that different applications built on top of the database will themselves have different consistency requirements, so it may be a good idea to provide multiple consistency models and flexibility in configuring these models for different types of operations in a database system. Many popular database system vendors allow their users to choose for the consistency model of their choice, even at the operations level. Some of those vendors offer true fine-grained consistency options, such as Microsoft Azure Cosmos DB, which offers even 5 models with gradually decreasing consistency constraints [@microsoft2022cosmosconsistency].

\todo{RabbitMQ consistency choices (quorum queues vs mirror queues) if time}

\paragraph{Immutability Changes Everything.}

Immutability naturally creates monotonicity, as the set of data—let it be either a payload or commands on this payload, like the `itemInserted`/`itemRemoved` example in subsection [@sec:calm] above—can only grow. Helland claims in his paper "Immutability Changes Everything" that "We need immutability to coordinate at a distance and we can afford immutability, as storage gets cheaper" [@helland2015immutability]. The latter statement is somewhat reminiscent of Moore's Law.

By designing a system to be append-only, and therefore monotonic, we receive all the benefits of coordination-free consistency. Append-only systems not only provide lower latency when replicated, but also better write performance at the local disk level. Many databases are equipped with a _write-ahead log_ (WAL) that records all the transaction to be executed to the database in advance. These write-ahead logs allow for reliable and high-speed appends, because records are appended immutably, atomically, and sequentially. The log contains virtually the truth about the entire database and allows to validate past and recent transactions (e.g., in the event of a crash), as well as time travel to previous states of the database, acting like a ledger. Even redo and undo operations are stored in this log. As shown in subsection [@sec:calm], replicated and distributed file systems depend on immutability to eliminate anomalies. By deriving aggregates from append-only logs of observed facts, consistency can be guaranteed to a certain degree[^tampering-logs]. From a particular perspective, a database is nothing more than such a large, derivative aggregate. Append-only structures also increase the safety of a system and thus support its fault-tolerance: if a system is limited to functional computations on immutable facts, operations become idempotent. Then the system does not become faulty due to failure and restart.

[^tampering-logs]: In large scale distributed systems, the consistency of such logs can still be victim to tampering and other security threats, as it is the case for blockchains (cf. subsection [@sec:blockchain-consensus]).

When building a distributed system and thinking about the consistency models, it is therefore useful to think about the nature of the data that this system will store and manage in advance. Table \ref{table:inside-vs-outside-data} helps in categorizing data into _inside data_ and _outside data_, as described by Helland [@helland2015immutability]. The latter is immutable and therefore allows for coordination-free consistency.

<!-- TODO What about LSM (Log Structured Merge trees) for append-only? -->

\begin{table}[h!]
    \caption{Categorization of inside vs outside data}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l l} 
        \toprule
         & \thead{Inside Data} & \thead{Outside Data} \\
        \midrule
        Changeable & Yes & No: Immutable \\
        Granularity & Relational field & Document, file, message or event \\
        Representation & Typically relational & Typically semi-structured \\
        Schema & Prescriptive & Descriptive \\
        Identity & No: Data by values & Yes: URL, document id, UUID... \\
        Versioning & No: Data by values & Versions may augment identity \\
        \bottomrule
    \end{tabularx}
    \label{table:inside-vs-outside-data}
\end{table}

\paragraph{Influence of the Network Infrastructure and Overall Architecture.}

Not only the use case, but also the capabilities of the infrastructure the system will be deployed onto (i.e., the network)  and the overall technical architecture play an important factor in the decision. Under certain circumstances, it is possible to provide high levels of consistency and yet low latency and availability, e.g., by using multiple or even nested layers of different consistency models and intelligent partitioning techniques.

The critique of the CAP theorem presented in the previous subsections allows for a more deliberate choice of consistency in practical systems, since several other properties can affect the actual requirements for consistency and dependability, often even more than the original theoretical properties of the CAP theorem. As an example, Google's distributed _NewSQL_ database system Spanner is in theory a CP class system [@corbett2013spanner]. It's design supports strong consistency with realtime clocks. In practice, however, things are different: given that the database is proprietary and runs on Google's own infrastructure, Google is in full control of every aspect of the system (including the clocks). The company can employ additional proactive strategies to mitigate network issues (such as predictive maintenance) and to reduce latency in Google's extremely widespread data center architecture. In addition, intelligent sharding techniques have been deployed (we discussed partitioning and sharding in subsection [@sec:partitioning]) that take advantage of this data center architecture. As a result, the system is highly available in practice (records to date even show availability of more than five nines (99.999 %) at the time of writing), and manifested network partitions are extremely rare. Eric Brewer, the author of the CAP theorem and now (at the time of writing) VP of infrastracture at Google, even claims that Spanner is technically CP but effectively CA [@brewer2017spanner]. We'll look at Spanner in more detail in section [@sec:previous-work]. It is important to realize that this is difficult in practice for open, self-managed distributed databases, or generally for smaller, less complex infrastructures, or when there is no control over the underlying network, as this requires a joint design of distributed algorithms and new network functions and protocols (cf. the Eris protocol in subsection [@sec:coordination-free-replication]). And after all, it is always a question of overall economic efficiency.

In addition, the actual choice of a replication protocol adds another layer of considerations that must be factored into the decision. As a rule of thumb, the more tightly coupled the replicated database system and infrastructure, the easier it is to ensure strong consistency without compromising latency and availability.

<!-- TODO further mitigation strategies (partial repl, horizontal scaling/sharding, improved network protocols)) -->

#### Further Cost Reduction Strategies {#sec:cost-reduction}

\paragraph{Service Decoupling.}

The additional cost of hardware can be mitigated by separating compute-intensive application services from the data store, as shown in figure \ref{fig:service-and-data-replicas}. As storage becomes cheaper (and, in the long term, compared to the costs of unavailability and failures, providing _zero marginal cost_), the data store can be deployed on multiple less powerful nodes with high storage capacities, while the application services are deployed on more powerful machines but with lower storage capacities and a much lower replication factor. If the services are designed to be stateless, replication of the services is only required for availability and load balancing, since these services do not need to be coordinated with each other.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/service-and-data-replicas.pdf}
  \caption[Seperated services and data store]{Service replication and data replication are different: while compute-intensive, stateless services can be deployed on powerful machines with a lower replication factor for load-balancing and high availability, data is replicated on more, but less powerful nodes for fault-tolerance and dependability.}
  \label{fig:service-and-data-replicas}
\end{figure}

\paragraph{Partial Replication.}

Not all data must be replicated, and not all with the same level of consistency. With partial replication, each replica holds only a subset of all data. This is often used for geo-replication: the data is replicated to nodes in close proximity to the respective clients. The replica subsets are distributed over multiple nodes and clusters in a way that in case of a node or cluster failure, the total set of data can be recovered from the various partial replicas. This is shown schematically in figure \ref{fig:partial-replication-sharding}. This can be achieved with sharding where the shard keys also correlate with data locality. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/partial-replication-sharding.pdf}
  \caption[Example scheme for partial replication]{Example scheme for partial replication. The same pattern can be applied intra-cluster (by load-balanced partial replication of shards) and inter-cluster (by data-locality-aware geo-replication of shards or even whole clusters).}
  \label{fig:partial-replication-sharding}
\end{figure}

\paragraph{Elastic Horizontal Scaling.}

With _elastic scaling_ (often referred to as _auto-scaling_), it is possible to maintain a consistently high throughput rate even as requests to the system increase. Elasticity means that new shard nodes are automatically provisioned on additional hardware resources and the load is balanced if a trend towards higher throughput in the long term is detected (and vice versa, if the trend is downwards, then superfluous resources are removed again). This requires constant monitoring of the system. With a lot of clients served this way, the costs for replication converge to marginal costs compared with the increased throughput and constant latency provided to clients. In append-only systems, this gets more complicated as sharding a single event stream means that it must be merged again on queries.

The impact of horizontal scaling on the overall system performance can be more significant than the choice of consistency model: even with eventual consistency, the system will never become consistent if the average write rate remains higher than the maximum throughput of the current replica shards, since the replica states will never converge even if the current primary node accepts such high write rates.

\paragraph{Smart Engineering.}

Since the network overhead can quickly become the most expensive part of an operation in a data store, it makes sense to send and execute operations in batches to reduce the share of the network overhead on the total processing time. The batch can be processed atomically, much like a transaction: either all or none of the operations in the batch are executed; however, it can also be executed optimistically: executing as many operations of the batch as possible. Both approaches come with a new cost: the risk of data loss if the batch is not completely written. If the node processing the batch fails over, it must compare what has already been written and what have been in the batch (which should have been logged in a write-ahead-log) if we want to provide clients with fire-and-forget behavior.

The batch behavior can be enforced by using a _buffer_ on the processing node. The buffer can not only reduce the overall network overhead, but also temporarily cushion exceptionally high write rates if the average rate remains below the maximum throughput (if the average rate stays over the maximum throughput rate for a longer period of time, the overall system will slow down and likely crash).

As we have shown in subsection [@sec:consistency-decisions], it makes sense to think beyond the application layer and to start questioning the network layer, in case you are in control of the underlying network architecture and protocols. Under certain circumstances, latency and availability can be kept high even with a high level of consistency, and even with transaction management, as we will show in subsection [@sec:coordination-free-replication].

Note that it is worth questioning the consistency model or the implementation itself before considering smart but complex engineering techniques to improve latency and availability while maintaining a strong consistency model (see subsection [@sec:consistency-decisions]).

<!--
### Error Correcting Codes

TODO only a short excursion
-->

### Theoretical Replication Protocols {#sec:replication-protocols}

Now that we have discussed dependability, consistency models, and the costs and trade-offs of replication, we can finally get more specific: the next subsections describe the different categories of replication protocols and follow with a discussion of relevant protocols.

#### Consensus Protocols {#sec:consensus-protocols}

We start looking into replication protocols that provide strong consistency, namely _consensus protocols_. The first research on consensus protocols was about concurrent memory access by multiple threads or processes on the same machine. The same principles apply to nodes in a distributed system writing to distributed memory or persistent storage, since the main issue is the agreement on a single value to be written. The first mention of a consensus algorithm in the distributed system literature was 1979 in a paper by Robert H. Thomas [@thomas1979majority].

Consensus protocols have been described to solve the _consensus problem_ [@lamport2005generalized]. The consensus problem describes the problem that a majority of nodes in a distributed system must agree on a single value. Consensus protocols describe how this majority can be achieved by specifying the necessary communication steps for coordinating between nodes, and between nodes and the client. All this happens in the context of fault-tolerance: nodes must agree on a common value even in the presence of failures, while providing a high level of availability without compromising on consistency. This can be summed up as follows: "a consensus protocol enables a system of $n$ asynchronous processes, some of which are faulty, to reach agreement [@bracha1983resilient]."

Consensus protocols typically have the following properties [@ongaro2014consensus]:

- They never return a wrong result under all faulty conditions (either fail-stop or even byzantine, depending on the protocol), including network partitioning, packet loss, duplication and reordering.
- A system is available as long as the majority of the nodes are operational
and can communicate with each other and with the clients.
- They do not depend on external timers to ensure the consistency of the logs.
- Under normal conditions, a write request can be served as soon as an absolute majority of the cluster has agreed on it; a minority of slow servers will not affect the overall performance of the system.
- Each process starts with some initial value; at the conclusion of the protocol all operational nodes must agree on the same value.

Following are some example applications where consensus is needed:

- Clock synchronisation,
- Google PageRank,
- smart power grids,
- cluster metadata management,
- service load-balancing,
- container orchestration (such as Kubernetes),
- key-value stores in general, 
- distributed ledgers and ultimatively the Blockchain (note that some of the consensus protocols here implement less strict consistency models, but are still called consensus protocols).

\paragraph{Synchronous vs. Asynchronous Consensus.}

Certain different variations of how to understand a consensus protocol appear in the literature. They differ in the assumed properties of the messaging system, in the type of errors allowed to the processes, and in the notion of what a solution is. Most notable is the differentiation between _consensus protocols for asynchronous and synchronous message-passing systems_. Fischer et al. have proven in the famous _FLP impossibility result_ (called after their authors) that a deterministic consensus algorithm for achieving consensus in a fully asynchronous distributed system is impossible if even a single node crashes in a fail-stop manner [@fischer1985impossibility]. They investigated deterministic protocols that always terminate within a finite number of steps and showed that "every protocol for this problem has the possibility of nontermination, even with only one faulty process." This is based on the failure detection problem discussed earlier in subsection [@sec:possible-faults]: in such a system, a crashed process cannot be distinguished from a very slow one. The FLP result is an important finding for theoretical considerations.

As pessimistic as this may sound, the FLP result can easily be mitigated in practice in one of two ways:

- Assume synchronicity in the protocol,
- Add nondeterminism to the protocol.

Bracha et al. consider protocols that may never terminate, "but this would occur with probability 0, and the expected termination time is finite. Therefore, they terminate within finite time with probability 1 under certain assumptions on the behavior of the system[^eventual-liveness]" [@bracha1983resilient]. This can be achieved by adding characteristics of randomness to the protocol, such as random retry timeouts for failed messages, or by the application of _unreliable failure detectors_ as in the _Chandra–Toueg consensus algorithm_ [@chandra1996unreliable]. Additionally, adding pseudo-synchronous behavior like in Raft (which is described in detail in section [@sec:raft]), where all the messaging goes in rounds in a bounded timeframe by enforcing a message enumeration and adding an upper bound for message time by using timeouts, removes the initial problem of asynchronous consensus.

[^eventual-liveness]: So, to be formally correct, we say that nodes in a consensus protocol _eventually_ agree on a value and an order.

\paragraph{Centralized vs. Decentralized Consensus.}

In _centralized consensus_, an additional authority is responsible for coordinating the nodes when a majority vote is pending. The _Zookeeper Atomic Broadcast_ (ZAB) protocol is one example for such a centralized protocol [@hunt2010zookeeper]. Such centralized protocols are problematic as the additional coordination service is the bottleneck—if the service can not be reached due to a failure or network partitioning, the whole system becomes unavailable—and must therefore be replicated with strong consistency, too.

In _distributed consensus_, a self-managed quorum is responsible for coordination. Such protocols do not rely on an additional service because the protocol is built in the replicated system itself. There are protocols with strong single-leader characteristics, where only a single node of the quorum is allowed to serve both read and write requests, and there are leader-less protocols, as well as combinations of both. Popular examples for distributed consensus protocols are _Paxos_ (subsection [@sec:paxos]) and Raft (section [@sec:raft]), the latter being the focus of this work.

Leader-less _decentralized consensus_ is the only option for consensus in highly decentralized multi-agent systems, such as _blockchains_. Decentralized consensus enables many actors to persist and share information securely and consistently without relying on a central authority or trusting other participants in the network.

\paragraph{Single-Value vs. Multi-Value Consensus.}

Another way to categorize consensus protocols is by the set of values to which the protocol refers:

In _single-value consensus protocols_, nodes agree on a single value. Those protocols are not designed to agree on a consistent ordering of operations. Originally, the Paxos (see subsection [@sec:paxos]) algorithm was described as a single-value consensus protocol. In the literature, the word consensus refers to single-value consensus in general.

In _multi-value consensus protocols_, the nodes also agree on a consistent ordering of the operations and values, forming a progressively-growing operation history. The correctness requirement is thereby two-fold: correct execution results for all requests and correct order of these requests. This may be achieved naively by running multiple iterations of a single-value consensus protocol that has been extended to include timestamps, but many optimizations of coordination and other considerations such as reconfiguration support can make multi-valued consensus protocols more efficient in practice. _Multi-Paxos_ is an example for such a protocol. 

\paragraph{Crash-Fault-Tolerant vs. Byzantine-Fault-Tolerant Consensus.}

Yet one more way to categorize consensus protocols is by fault-tolerance. As described in subsection [@sec:safety-reliability], a consensus protocol is called $k$-fault tolerant if in the presence of up to $k$ faulty nodes it reaches consensus with probability 1.

A protocol that solves the consensus problem in the face of faulty nodes must guarantees the following properties (extending the liveness and safety properties as described in subsection [@#sec:safety-reliability]): 

- **Termination**: Every non-faulty node eventually decides on some value,
- **Integrity**: If all non-faulty nodes proposed the same value $y$, then any non-faulty node must decide $y$,
- **Validity**: The agreed-upon value must be the same as the initially proposed value,
- **Agreement**: Every non-faulty node must agree on the same value.

<!-- ============================================== -->
<!-- Crash-Fault Tolerant (CFT) Consensus Protocols -->
<!-- ============================================== -->

\paragraph{Crash-Fault Tolerant (CFT) Consensus Protocols.}

A _crash-fault tolerant_ (CFT) consensus protocol provides fault-tolerance for fail-stop failures, but not for byzantine failures. We have shown in subsection [@sec:possible-faults] that for a system to tolerate up to $k$ faults, at least $k + 1$ nodes are required, but this only accounts for a read operation perspective. For such a system to still be able to reach consensus on write operations, even more nodes are required: $\left \lceil (n + 1)/2 \right \rceil$ correct nodes, thus more than half of all nodes, are necessary and sufficient to reach agreement [@bracha1983resilient]. We call this number a _quorum_. Based on this number, we know how big a cluster must be to tolerate $k$ crashed nodes: $n \geq 2k + 1$ where $n$ the size of the cluster.

A consensus protocol is fault-tolerant by masking failures. But in practice, they also apply failure detection: to detect a crash-failure, nodes in a quorum typically send periodic heartbeats. If for some node no heartbeat was received for a certain timeout period, the protocol assumes that this node is dead and removes it from the quorum. In the literature, there are more sophisticated approaches to failure detection, but for distributed databases, heartbeats are most common in practice and easy to implement. 

<!-- ================================================== -->
<!-- Byzantine-Fault Tolerant (BFT) Consensus Protocols -->
<!-- ================================================== -->

\paragraph{Byzantine-Fault Tolerant (BFT) Consensus Protocols.}

\epigraph{It is not sufficient that everyone knows X. We also need everyone to know that everyone knows X, and that everyone knows that everyone knows that everyone knows X — which, as in the Byzantine Generals’ problem, is the classic hard problem of distributed data processing.}{--- \textup{James A. Donald}}

A _byzantine-fault tolerant_ (BFT) consensus protocol provides fault-tolerance for byzantine failures. The problem of byzantine faults and byzantine consensus was first described by Leslie Lamport in the _Byzantine Generals Problem_ [@lamport1982byzantine]. It is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a system architect. 

The Byzantine Generals Problem describes a situation where a number of generals besiege a town with their armies, surrounding this town to plan a concerted attack. They can only win this attack if they all attack together at once, so they need to agree on the time of the attack. They can also decide together to retreat. They also assume that some of the generals are disloyal and will send out false commands. Due to the distance between the generals, they can only communicate with each other via messengers on horseback, so there is no way of verifying the authenticity of such a message. Lamport et al. have shown that the problem can only have a solution if more than two-thirds of the generals are loyal. Figure \ref{fig:byzantine-generals} illustrates the byzantine problem for three nodes, which can not be solved.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/byzantine-generals.pdf}
  \caption[Byzantine generals problem with three nodes]{The byzantine generals problem can not be solved with only three nodes}
  \label{fig:byzantine-generals}
\end{figure}

For read requests, a cluster must have at least $2k + 1$ nodes so a majority of nodes can still respond with the non-arbitrary values when $k$ nodes are faulty, as shown in subsection [@sec:possible-faults]. But to be byzantine-fault tolerant BFT for write operations, we need more nodes to be able to agree on the correct value. The quorum must be significantly larger than that for crash-fault tolerance: for any consensus protocol to be byzantine-fault tolerant, a quorum of $\left \lceil (2n + 1)/3 \right \rceil$ correct nodes are necessary and sufficient to reach agreement. Therefore, to tolerate up to $k$ faulty nodes, a cluster must have at least $n$ nodes with $n \geq 3k + 1$. This fundamental result was first proved by Pease, Lamport et al. [@pease1980faults] and later adapted to the BFT consensus framework [@bracha1983resilient].

Every byzantine-fault tolerant consensus protocol is also fail-stop-fault tolerant [@xiao2020survey]). Blockchain protocols generally belong to the class of BFT consensus algorithms, as crashed nodes are not a problem in general, but faulty values are. We describe this briefly in subsection [@sec:blockchain-consensus].

We won't go into much details here, as this work is limited to a problem scope where byzantine faults are rare (but not yet excluded).

\paragraph{Weighted quorums.}

Some consensus algorithms, like the Proof-of-Stake algorithm used for certain blockchains and cryptocurrency tokens, allow for weighted quorums. As a rule in general consensus protocols, the votes of all quorum members have the same weight in a vote. With weighted quorums, however, there may be cluster members whose vote is more important. Whether a system is $k$-fault-tolerant or not is now judged by the sum of the relative weights rather than the number of nodes. To perform a write operation, an agreement of the majority of the nodes is now no longer mandatory, if among them there are nodes whose vote has relatively more weight. 

\paragraph{Network reconfigurations.}

If a node is removed from the cluster because it is faulty or has not responded for a certain period of time, and is then restored again, it attempts to rejoin the cluster. Also, when the system is scaled horizontally by adding new nodes, these nodes can join the cluster if the load balancer assigns them to do so, i.e., for a new shard. In the case of network partitioning, many nodes can no longer be reached, but the system should remain available. Or due to planned maintenance, the hardware of the nodes is upgraded, or the IP addresses of the nodes change if not statically assigned. In the case of blockchains, new nodes are added and removed at very short intervals. In all of these cases, the network is reconfigured, and generally clients are not expected to notice, i.e., the system remains available and consistent while the network is reconfigured, and latency does not degrade. The consensus protocols used in practice generally enable such seamless network reconfigurations, especially the decentralized protocols.

<!-- total order broadcast protocol, which is known to be equivalent the consensus problem -->

#### State Machine Replication {#sec:state-machine-replication}

The _state machine replication_ (SMR) protocol describes a protocol with strong consistency that is based on the simple idea of a _deterministic state machine_ and a set of operations[^operation-command] that can be executed on it. Every node of the distributed system represents such a state machine, and operations can be requested by clients of the distributed system like they would request them on a non-replicated state machine on a single server. A central idea of SMR is a serializable write-ahead log where all requested operations are written to in the order of their proposed execution before the actual execution. This log is replicated between all nodes of such a cluster, therefore the core protocol of SMR is referred to as _log replication_. An entry is only written to the log after a majority of nodes agreed on it. These operations will be executed on the state by the individual state machines in the order that they appear in that log. All these operations are functional and side-effect free, and since the state machine is deterministic, all the servers will produce the same sequences of states, resulting in the states of all nodes always being consistent. This allows for an reduction of the state of a distributed system and its individual nodes to an append-only log. The SMR approach is illustrated in figure \ref{fig:state-machine}.

[^operation-command]: We will use the terms _operation_ and _command_ interchangeably throughout this work.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/state-machine.pdf}
  \caption[Messaging between clients and cluster in state machine replication]{Illustration of the messaging between clients and cluster in the state machine replication approach}
  \label{fig:state-machine}
\end{figure}

State machine replication is the protocol of choice in many popular services and the subject of numerous academic studies, originating in Leslie Lamport's early work on clock synchronization in distributed systems [@lamport1978time]. Lamport shows that "a distributed system can be described as a particular sequential state machine that is implemented with a network of processors. The ability to totally order the input requests leads immediately to an algorithm to implement an arbitrary state machine by a network of processors, and hence to implement any distributed system". In the original paper, the SMR approach wasn't discussed in the face of failures: "The problem of failure is a difficult one, and it is beyond the scope of this paper to discuss it in any detail." The first formal description of the SMR approach to provide fault-tolerance was presented by Schneider [@schneider1990statemachine] in 1990. 

In the previous subsection, we investigated the consensus problem, consensus protocols and their various categories. In past years, the relationship between SMR and consensus has been the subject of numerous studies. Consensus is the basis for state machine replication and its main component. State machine replication introduces the perspective of a consistent state. Log replication—its core component—in turn extends decentralized multi-value consensus protocols by introducing the perspective of a consistent log. This consistent log and state result from consensus on both operations and the order of these operations to execute: all nodes must agree on the same state, even if some nodes crash or disconnect, which requires consensus for each command, but also to execute these commands in the same order, otherwise different replicas may end up in a different final state. We could say that SMR is an approach to solving the consensus problem from an engineering perspective, while consensus protocols give a theoretical perspective on this problem [@lamport2005generalized].

The state machine can be formally described by [@lamport1978time]

- a set $C$ of possible commands, 
- a set $S$ of possible states, 
- and a state transition function $\delta: C \times S \to S$ with $delta(c, s) = s'$, $c \in C$ and $s, s' \in S$.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/state-machine-replication.pdf}
  \caption[The log replication mechanism]{Illustration of the log replication mechanism in the state machine replication approach}
  \label{fig:state-machine-replication}
\end{figure}

The SMR approach sits in the client-server realm, where it tries to emulate a single machine towards a set of clients. We can extend safety and liveness in this sense from the point of view of the log:

- **Safety**: All nodes store the same prefix in their logs, i.e., if a node appends $a$ at the index $i$ to its log, then no other node will append a different command $b \neq a$ at this index $i$ to its log.

- **Liveness**: All nodes will eventually apply a command issued by a client, i.e., if a client issues a command $c$, then eventually all nodes will have a log entry $c$ appended at some index $j$ in the log and all previous indexes $i < j$ in the log will be set.

For safety, this requires the following guarantees [@xiao2020survey]:

- All server nodes start with the same initial state,
- _Total-order broadcast_: all nodes receive the same sequence of requests in the order they were issued by clients,
- All nodes receiving the same request will always output the same execution result and end up in the same state.

\paragraph{Single Leader Election.}

In state machine replication, there is commonly a concept of a single, strong leader. A strong leader node is the only instance in a SMR protocol that is allowed to communicate with the client and to coordinate writes—and often also reads—between replicas. This ensures linearizability, as the single leader can ensure that commands will be ordered in the same order as a client has requested. The leader must be somehow decided on between the replicas, and this must happen with strong consistency guarantees. This process is called _leader election_. In the same way as consensus and SMR agree on a single value or a order of commands, they must agree on a leader decision, i.e. through another instance of consensus. The FLP impossibility result (as described in subsection [@sec:consensus-protocols]) hence implies that a reliable leader election protocol must use either randomness or real time—for example, by using timeouts.

\paragraph{The Generation Clock.}

State machine replication mitigates the FLP impossibility problem by adding pseudo-synchronous behavior through timeouts and a _generation clock_. If no progress is made in deciding on the next command in a certain amount of time, or when a new leader is elected due to a leader failure or timeout, a new round of the protocol begins, in which the steps may be repeated again. This rounds are marked by a generation clock, sometimes refered to as a _term_ or _epoch_. The generation clock is allowed to only increase monotonically. The clock ensures that in case of failures and network partitions, once a node rejoins a quorum, messages issued by that node (i.e. a leader node) that are associated with an outdated generation clock are ignored, and the node can synchronize itself again with the new clock. In case of network partitioning, meanwhile committed entries of the partitioned nodes may be dropped to restore consistency across the whole distributed system. The generation clock is persisted together with the log entries, so replica nodes can use this information to find conflicting entries in their log after a fail over.

\paragraph{The High-Water Mark.}

In a system with no failures, the commands could be discarded after being applied to the state machine. But in the face of failures, the system must know what has been successfully committed and what is still pending, to restore replica states from the log[^logless]. For strongly consistent replies to read requests, any cluster node is only allowed to return a state based on the latest log entry that was successfully replicated to a quorum. The _high-water mark_ is an index into the log file that marks this latest log entry. During instances of the replication protocol, the high-water mark is passed to replicas. All servers in the cluster should only allow reads to clients that reflect updates that are below the high-water mark.

[^logless] There are approaches to consistent, logless SMR protocols that still provide safety and liveness, such as proposed by Skrzypzcak et al. [@skrzypzcak2020towards]: instead of agreeing on a sequence of commands to build up a consistent log, nodes in logless SMR agree directly on the sequence of state machine states.

The current state can always be reconstructed by applying the committed portion of the log up to the high-water mark:

$$\delta(log_{\mathrm{hwm}}, s) = \delta_{cmd_i} \circ \dots \circ \delta_{cmd_0} (s)$$

It is theoretically possible to replay only a part of the log to achieve a time-travel query, similar to event sourcing. In practice, the command log grows unbounded and depending on the state that is managed, such time-travel queries are too expensive, yet there are event sourcing systems built on replicated state machines where the state is an optimized append-only data structure holding the events of interest for sourcing, while the command log contains more events and operations (such as node management and other metadata commands that are not actual payload). The unbounded nature of the log is mitigated in different ways: one way is to keep the log transient in-memory instead of persisting it on disk. This makes fail over more complicated, but it is possible under circumstances if some features such as network reconfiguration support are not required. 

\paragraph{Snapshotting and State Transfer.}

Another way to keep the log size bounded is to apply _log compaction_. _Snapshots_ of the current state are created periodically or as a reaction to certain events or commands in the log. These snapshots also hold an index to the latest log entry that was included to derive this state. All log entries to this point are then removed from the log. We illustrate this in figure \ref{fig:log-compaction}. When a new node joins the cluster, instead of all the log entries, the latest snapshot and all additional committed log entries since then are sent to this new node.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.65\textwidth]{images/log-compaction.pdf}
  \caption[Compaction of committed log entries]{Compaction of committed log entries by creation of a state snapshot}
  \label{fig:log-compaction}
\end{figure}

\paragraph{Log Truncation.}

When a server joins the cluster after crash/restart, there is always a possibility of having some conflicting entries in its log. So whenever a server joins the cluster, it checks with the leader of the cluster to know which entries in the log are potentially conflicting. It then truncates the log to the point where entries match with the leader, and then updates the log with the subsequent entries to ensure its log matches the rest of the cluster.

This can be conflicting with log compaction. If the latest log entry that happens to be valid after truncation is already compacted, the whole state must be transfered, which is more expensive then trasmitting only a diff of the log entries.

\paragraph{Network Reconfiguration.}

Network reconfiguration support in SMR is evenly important to those in Consensus, see subsection [@sec:consensus-protocols]. In addition, in SMR, a network reconfiguration also includes a state transfer, where a newly joined or rejoined node receives and installs the latest state snapshot.

\paragraph{Failure Detection.}

In SMR, failures are generally masked. But to provide a single leader, a failure detection mechanism is neccessary to detect a leader failure, to prevent the system from being unavailable during leader crashes. In most SMR protocols, this is realized with heartbeats and a timeout. Once a node does not receive a heartbeat from the leader anymore in a certain timeframe, it assumes that the leader crashed and starts a new leader election, which also triggers an increment of the generation clock.

\paragraph{Relation to the Consensus Problem.}

State machine replication uses consensus under the hood to agree on a single value in a round of voting. But additionally to consensus on a single value, it adds other properties to the protocol, especially the ordering mechanism. Although there is also a consistent ordering in multi-value consensus, the state machine approach is different: it approaches the problem from the perspective of the state, not the consensus. Or, as Antoniadis et al. note: "Solving consensus is one step away from implementing state machine replication" [@antoniadis2018state]. They have shown that SMR is more expensive than consensus from a complexity point of view, and even under synchrony conditions, no SMR algorithm can guarantee bounded response times compared to Consensus.

\paragraph{Relation to Weaker Consistency Models}.

It appears that the state can be seen as a derived non-monotonic aggregate, while the commands in the log are trivial facts and operations on it. From this point of view, if the state appears to be or can be designed as a monotonic aggregate, weaker consistent replication protocols are more suitable, such as optimistic replication (see subsections [@sec:calm] and [@sec:optimistic-replication]).


\paragraph{Byzantine-Fault Tolerance}.

All these safety and liveness properties described in this subsection do not apply to byzantine failures, i.e., they only apply to _honest_ nodes. Basic state machine replication is only crash-fault tolerant, while there are also byzantine-fault tolerant state machine replication approaches, such as _Practical Byzantine Fault Tolerance_ (PBFT) [@castro1999practical]. Since byzantine-fault tolerance is not the focus of this paper, we will not discuss it in detail here, but refer to the relevant literature [@distler2021byzantine].

\todo{Relation to Atomic Broadcasts?}

<!--
SMR requires an implementation of a total order broadcast protocol, which is known to be equivalent to the consensus problem.

in systems with crash failures, atomic broadcast and consensus are equivalent problems. 
A value can be proposed by a process for consensus by atomically broadcasting it, and a process can decide a value by selecting the value of the first message which it atomically receives. Thus, consensus can be reduced to atomic broadcast. [@chandra1996unreliable]
Conversely, a group of participants can atomically broadcast messages by achieving consensus regarding the first message to be received, followed by achieving consensus on the next message, and so forth until all the messages have been received. Thus, atomic broadcast reduces to consensus. This was demonstrated more formally and in greater detail by Xavier Défago, et al.[https://doi.org/10.1145%2F1041680.1041682]

The Chandra-Toueg algorithm[@chandra1996unreliable] is a consensus-based solution to atomic broadcast.

The Zookeeper Atomic Broadcast (ZAB) protocol is the basic building block for Apache ZooKeeper, a fault-tolerant distributed coordination service which underpins Hadoop and many other important distributed systems.[8][9]

-->

<!--
https://en.wikipedia.org/wiki/Atomic_broadcast

In systems with crash failures, atomic broadcast and consensus are equivalent problems.[@chandra1996unreliable]

A value can be proposed by a process for consensus by atomically broadcasting it, and a process can decide a value by selecting the value of the first message which it atomically receives. Thus, consensus can be reduced to atomic broadcast.

Atomic Broadcast is equivalent to
State Machine Replication (SMR):
üAny SMR algorithm can be used to implement
Atomic Broadcast
üAny Atomic Broadcast algorithm can be used to
implement SMR

So, in crash-fault systems, atomic boradcast equiv consensus equiv SMR

Conversely, a group of participants can atomically broadcast messages by achieving consensus regarding the first message to be received, followed by achieving consensus on the next message, and so forth until all the messages have been received. Thus, atomic broadcast reduces to consensus. This was demonstrated more formally and in greater detail by Xavier Défago, et al.[2]


https://arxiv.org/pdf/1710.07845.pdf

Total order broadcast primitives are a critical component for
the construction of fault-tolerant applications based upon active replication, aka state machine replication [8,18]. The
primitive guarantees that messages sent to a set of processe
s
are delivered, in their turn, by all the processes of the set
in the same total order. A possible way of implementing
total order broadcasts is through multiple executions of a
consensus algorithm. Thus, the performance of the total order broadcast is directly dependent on the performance of
the consensus algorithm. 

-->

<!--
#### Other Kinds of Protocols

- Master-Slave
- Active-Passive (any difference to master-slave?^)
- Primary-Copy
- TODO Mention them here or just below?
-->

#### Read One / Write All (ROWA)

_Read one / write all_ is the simplest, yet weakest approach to strongly consistent replication. Its two properties are:

- **Read one**: Any replica node can answer a read request, even if all other nodes failed.
- **Write all**: All replicas can be written to from any client, and the write is only confirmed successfully if it is replicated on all replica nodes.

In ROWA, the cluster is immediately unavailable for writes once a single node crashes. In addition, the write latency grows with each replica in the cluster, as every node participates in writing and must acknowledge their writes. The latency is therefore at least as high as the longest latency of any node, rendering slow nodes as a bottleneck. In quorum-based replication, as we presented in consensus and state machine replication, a single unresponsive or slow node will never be a bottleneck if a quorum of responsive nodes can be achieved.

In contrast, for read operations, the availability is very high and the latency low since a data object can be retrieved from any replica without delay.

#### Primary-Copy Replication {#sec:primary-copy}

When we allow for transactions instead of only single values, replication with ROWA becomes more unstable and prone to problems: as the workload scales up, it becomes exponentially slower and prone to deadlocks or reconciliations, since all nodes must coordinate with each other pair-whise. The _primary-copy replication_ approach was introduced in 1996 to reduce this problem [@gray1996dangers]. There are a variety of primary copy replication approaches, ranging from simple and weak approaches that make some trade-offs in fault tolerance, such as continuous backups of relational databases, to strongly consistent approaches that include state machines and atomic transfers.

In a primary-copy replication setup, there is one node holding a primary-copy of the data, while other nodes provide read-only backups of the data. Unlike state machine replication, primary-copy replication does not replicate the commands themselves, but the results of their execution—which is the diff of the state before and after the execution [@wiesmann2005comparison]. This also means that in all cases commands are first executed on a primary copy of the data before their results are replicated. The disadvantage of this approach is that this does not ensure strong consistency, but only sequential consistency of distributed copies: what is written, can be read immediately only on the node holding the primary copy, while it appears on the replicas with a delay, but at least in the same order. This is a reason why there are applications of primary-copy that prevent reading from the replicas and only use them as backups for fault-tolerance, which re-introduces strong consistency as long as the primary copy does not fail (when this happens, rollback strategies must be in place, which is not always possible and error-prone). This is done, for example, in the _ZooKeeper Atomic Broadcast_ protocol (see subsection [@sec:zookeeper] for details).

Compared to ROWA, this system can tolerate failures, at least those of the backups. But, primary-copy nodes can easily become a bottleneck in the system. It does not scale with the number of nodes, similar to state machine replication and consensus. What happens if the node serving a primary-copy fails? Failure detection mechanisms are required to detect such situations and subsequently select an alternate primary-copy server (similar to the leader election problem of state machine replication), creating the risk of multiple primary-copies when different network partitions occur; a problem that is mitigated in some protocols. In addition, a failure of a primary-copy can cause loss of data not yet replicated. Until the instantiation of the backup, the system may also unaivable to the user.

Primary-copy is often used in RDBMS to provide continously updated hot backups. There are different approaches in primary-copy replication to handle transactions. One approach involves performing transactions atomically on the primary copy first, before passing the result of the transaction to the replicas. 

\paragraph{Primary-Copy with Logs and State Machines.}

It is possible to implement the primary-copy approach with a replicated log and state machines. Instead of the actual commands, this log holds the outcomes of the command executions, but still in a strict sequential ordering. The replication is achieved after the execution of a client request by the primary copy by writing the state diff to a replicated log, which can happen in a similar same way as in state machine replication by applying consensus respectively total order broadcasts. This is illustrated in figure \ref{fig:primary-copy-state-machine}. To achieve linearizability for strong consistency, the effects of writes should not be made visible to clients until the log entries have been committed to a quorum, which is far from trivial to implement. In addition, the primary copy should also contain the clients' responses in the log entries so that the backup servers can return the same response if the clients retry. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.75\textwidth]{images/primary-copy-state-machine.pdf}
  \caption[State machine replication vs primary-copy replication]{State machine replication vs primary-copy replication. a) In state machine replication, the replicated log contains the actual commands and are consistently replicated to the logs before they are executed on the state machines. b) In primary-copy replication with state machines, the primary copy executes the command before the replication starts. After execution, it replicates the outcome of the execution, which is the diff between the previous and the new state, to the replicated logs, from where it is applied to the state machines.}
  \label{fig:primary-copy-state-machine}
\end{figure}

#### Active Replication {#sec:active-replication}

In _active replication_, the clients are responsible to ensure consistency, therefore they participate actively in the protocol. Once a client issues a request, it must send this requests to all the replicas it wants the data to be replicated to, in contrast to _passive replication_, where only a single node receives the request and then takes care of the replication (such as in the other protocols shown so far). All replicas are equivalent, there is no leader node. The replicas take the requests in sequential order and execute them. After the execution, each individual node replies to the client. There is no direct communication between replicas, as all the coordination happens through the client. The client decides if it accepts the result of the write (i.e. one node, a quorum or all nodes acknowledged it), which also allows the client to compare the results to cope with byzantine errors (up to $n/2 -1$, see subsection [@sec:possible-faults] for reference). All dependability, consistency and performance properties therefore depend on how the clients act, e.g., if they wait for all nodes to acknowledge a write, the slowest replica is the bottleneck.

This approach does not provide strong consistency, as the replicas immediately execute a command once they receive it, and this can happen at different points in time, depending on the individual latency and load of the nodes. Moreover, multiple clients may invoke the system simultaneously, and even if they do not write at the same time, the order of their requests to each node can only be guaranteed to follow the program order, ensuring at least sequential consistency (cf. the sequencing in figure \ref{fig:consistency-ordering-sequential} of subsection [@sec:consistency]). Unless a client waits for all nodes to acknowledge a write, sequential consistency is also not guaranteed, as some intermediate writes may never have been performed by some nodes.

#### Optimistic Replication {#sec:optimistic-replication}

There are many use cases where the cost of replication is too high when synchronization between nodes in a replica set is required, especially for real-time live collaboration applications or applications that cannot be guaranteed to be online all the time (such as mobile apps). In these cases, the consistency requirements should be attenuated to eventual consistency so that the application can be designed in a non-blocking manner. One approach to solving problems of this scope is _optimistic replication_, as described by Shapiro et al. [@shapiro2005optimistic]. In optimistic replication, any client managing a replicated state is allowed to read or update the local replica at any time.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/crdts-optimistic-replication.pdf}
  \caption[Schematic communication model in optimistic replication]{Schematic communication model between the nodes in optimistic replication for a single operation. A write is acknowledged immediately and non-blocking, without synchronization with other nodes. It is then eventually propagated to other nodes. In the event of a write conflict, the conflict is automatically resolved in the background using either a decentralized protocol or a centralized entity. Once the conflict is resolved, the actual agreed value is written back to the node's internal state and acknowledged.}
  \label{fig:crdts-optimistic-replication}
\end{figure}

Such local updates are only tentative and their final outcome could change, as they may conflict with an update from another client. In optimistic replication, conflicts should not be visible to the user (since conflict resolution requiring human intervention would compromise consistency, which is undesirable in the intended use cases), so they must be resolved in the background, either by a decentralized protocol or a central server instance. The replicas are allowed to differ in the meantime, but they are expected to eventually become consistent. Figure \ref{fig:crdts-optimistic-replication} illustrates this behavior. Note that it only illustrates it for a single operation. In case of multiple operations, reordering the operations is also part of the conflict resolution, depending on the protocol.

Optimistic replication is especially suitable for problems that meet the monotonicity requirements as described in the CALM theorem (cf. subsection [@sec:calm]), as this class of problems requires no conflict resolution at all.

\todo{Describe steps of OR: server/decentralized protocol defines order of events...}

\todo{Possible to use CRDTs for chronicledb TAB+ tree?}

\paragraph{Conflict-Free Replicated Data Types.}

An example of optimistic replication for strong eventual consistency are _conflict-free replicated data types_ (CRDT). CRDTs were introduced by Shapiro et al. and describe data types with a data schema and operations on it that will be executed in replicated environments to ensure that objects always converge to a final state that is consistent across replicas [@shapiro2009crdts]. CRDTs ensure this by requiring that all operations must be designed to be conflict-free, since CRDTs are intended for use in decentralized architectures where operation reordering is difficult to achieve. Therefore, any operation must be both _idempotent_ (the same operation applied multiple times will result in the same outcome) and _commutative_ (the order of operations does not influence the result of the chained operations), which also requires them to be free of side effects [^secro].

CRDTs take the consistency problem onto the level of data structures. They make use of optimistic replication to allow for such operations without the need of replicas to coordinate with each other (often allowing updates to be executed even offline) by relying on merge strategies to resolve consistency issues and conflicts, once the replicas synchronize again. They are designed in a way that the overall state resolves automatically, becoming eventually consistent. Due to the nature of the operations to be idempotent, CRDTs can only be used for a specific set of applications, such as distributed in-memory key-value stores like Redis, which is commonly used for caching [@redis2022crdts]. Another common set of use cases are realtime collaborative systems, such as live document editing (relying on suitable data structures to hold the characters with good time complexity properties for both random inserts and reads, such as an _AVL tree_ [@adelsonvelskii1963algorithm]) [@shapiro2009commutative]. CRDTs are not suitable for append-only logs, streams or event stores, as designing an operation to append a record or event with idempotency and commutativeness is too expensive.

[^secro]: Recent research is trying to find such a replicated data type that does not require operations to be commutative. One approach describes _Strong Eventually Consistent Replicated Objects_ (SECROs), aiming to build such a data type with the same dependability properties as CRDTs [@de2019generic]. The authors achieve this by ensuring a total order of operations across all replicas, but still without synchronisation between the replicas: they show that it is possible to order the operations asynchronously.

CRDTs are not the only way to achieve optimistic replication. Before CRDTs, _operational transformations_ (OT) were the default approach for solving these kinds of problems, at least in academia. Due to their complexity and the difficulty to implement them and also to write formal proofs for different sets of operations needed in practical applications [@li2010admissibility], they were often replaced by alternative approaches like CRDTs, so we will not discuss them in detail in this work.

\todo{Grammar checking}
In theory, CRDTs are designed for decentralized systems in which there is no central authority to decide the end state. In practice, there is oftentimes a central instance, such as in SaaS offerings on the web. Decentralized conflict resolution is no longer a requirement for those systems, so CRDTs could be too heavy-weight. Moving the conflict resolution to a central instance (which actually could be a cluster of nodes with stronger consistency guarentees) reduces complexity in the implementation of optimistic replication. This is actually the case in multiplayer video games—especially massive multiplayer online games (MMOGs). They introduce a class of problems that need techniques to synchronize at least a partial state between a lot of clients. Strong consistency would cause this games to experience bad performance drops, since a game's client wouldn't be able to continue until it's state is consistent across all subscribers to that state, so the way to go is eventual consistency. One can learn a lot from these approaches and adopt it to other real-time applications, such as Figma did for their collaborative design tool, comprehensively described in a blog post [@figma2019multiplayer].

\todo{Work finished up to this point.}

---

#### Coordination-Free Replication {#sec:coordination-free-replication}

TODO refer to [@sec:calm] and https://www.microsoft.com/en-us/research/publication/eris-coordination-free-consistent-transactions-using-network-multi-sequencing/ [@li2017eris]

and [@sec:consistency-decisions] for being in control of the network infrastructure. (This means you can't just build this eris thing on a regular TCP network...) so, this of course subject of research of the big cloud vendors / datacenter owners (AWS, Google, Microsoft)

Eris from microsoft research

"The Eris transaction processing system achieves high performance through a new division of responsibility between
three parts. An in-network concurrency control primitive,
multi-sequenced groupcast, establishes a consistent order of
message delivery across shards, but does not ensure atomic
or reliable delivery. The latter guarantees are provided by
the Eris protocol, which makes sure that transactions are processed by all participant shards, or none at all. In combination,
these allow linearizable execution of independent transactions,
which make up a substantial part of many workloads. For
other workloads, a general transaction layer builds arbitrary
transactions out of multiple independent transactions.
The net result of this approach is that Eris can execute
independent transactions without any coordination
... linearizable...
Eris achieves strongly consistent, fault-tolerant,
transactional storage with overhead within 10% compared to
a system that provides no such guarantees."

Eris uses a quorum-based protocol to maintain safety 

Eris clients send independent transactions directly to the replicas in the affected
shards using multi-sequenced groupcast and wait for replies
from a majority quorum from each shard

"Unifying Replication and Transaction Coordination. Traditional layered designs use separate protocols for
atomic commitment of transactions across shards and for replication of operations within an individual shard. While this
separation provides modularity, it has been recently observed
that it leads to redundant coordination between the two layers [66]. Protocols that integrate cross-shard coordination and
intra-shard replication into a unified protocol have been able
to achieve higher throughput and lower latency [38, 48, 66].
This approach integrates particularly well with Eris’s innetwork concurrency control. Because requests are sequenced
by the network, each individual replica in a shard can independently process requests in the same order. As a result, in
the common case Eris can execute independent transactions
in a single round trip, without requiring either cross-shard or
intra-shard coordination."

TODO how does eris even avoid coordination between replicas inside of a shard??

answer: the sequencer in the network layer actually orders the stuff and then sends the commits to every replica in the shards, and they directly ack to the client, not to the DL. The DLs send the actual transaction results.
Then, the designated learner is responsible form synchronous execution of it's part of the transaction, while the followers log it and execute it later async. This is similar to NOPaxos. (So, no acks and not coordination between replicas? How is this secure? Is there a TLA+ spec for this? TODO check TLA+ for NOPaxos or Eris)
The authors write this:

"Eris must be resilient to replica failures (in particular, DL
failures)
In Eris, failure of the DL is handled entirely within the shard by a
protocol similar in spirit to standard leader change protocols"

", we
introduce a novel element to the Eris architecture: the Failure
Coordinator (FC). The FC is a service that coordinates with
the replicas to recover consistently from packet drops and
sequencer failures. The FC must be replicated using standard
means [39, 43, 51] to remain available. "

is the network protocol a bottleneck? Not additional, as if the network crashes, the whole thing crashes anyway. 

TODO what is then with partition tolerance???:
"Eris must be resilient to ... network anomalies"

\begin{figure}[h]
  \centering
  \includegraphics[width=0.7\textwidth]{images/eris-coordination.pdf}
  \caption[The coordination-free transaction and replication protocol Eris]{In the coordination-free transaction and replication protocol Eris, communication happens in a single round-trip in the normal case, while the transactions are consistently ordered by a sequencer on the network layer}
  \label{fig:eris-coordination}
\end{figure}

#### Chain Replication

"Chain replication is a new approach to coordinating
clusters of fail-stop storage servers. The approach is
intended for supporting large-scale storage services
that exhibit high throughput and availability without sacrificing strong consistency guarantees." [@van2004chain]

There are deviations from the original, strong-consistency case, such as ChainReaction, which provides causal+ consistency (see TODO ref to paragraph) and geo-replication, and is able to leverage the presence of multiple replicas to distribute the load of read requests [@almeida2013chainreaction].

"Allows to build a distributed system without external cluster management process"

TODO Comparison with primary-copy and state machine approaches

TODO see https://medium.com/coinmonks/chain-replication-how-to-build-an-effective-kv-storage-part-1-2-b0ce10d5afc3

In this subsection, we have presented common theoretical replication protocols. We sum them up in table \ref{table:replication-protocols-consistency-levels}. In the next subsection, we take a look at practical variants of these protocols as they are implemented in real-life applications.

\begin{table}[h!]
    \caption{Consistency levels of the presented replication protocols}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | l | l | l } 
        \toprule
         \thead{Protocol} & \thead{Consistency \\Level} & \thead{Write \\Latency} & \thead{Availability} \\
        \midrule
        Consensus & Strong & High & High \\
        State Machine & Strong & High & High \\
        ROWA & Strong & Highest & Lowest \\
        Primary-Copy & Sequential-Strong & Low–High & Low–High \\
        Active & Up to Sequential & Low–High & Low–Highest \\
        Optimistic & Eventual & Low & Highest \\
        \makecell[r]{Coordination-Free* \\[-4pt] \scriptsize *Only useable in rare cases} & Strong & Medium & High \\
        Chain & ? & ? & ? \\
        \bottomrule
    \end{tabularx}
    \label{table:replication-protocols-consistency-levels}
\end{table}

### Practical Replication Protocols

This subsection provides an overview of concrete, practical replication protocols to the reader. Some of them are used in production for years, while others are subject of academic studies only.

The [following subchapter](#sec:raft) will then spend a closer look on Raft, a consensus-based state machine replication protocol. The author has chosen to use Raft for replication of the event store which is the subject of this thesis.

#### Paxos {#sec:paxos}

One of the most popular distributed consensus algorithms is the Paxos consensus algoritm. It has been the de facto standard for distributed consensus in production systems for over two decades and is often used as a synonym for distributed consensus. Paxos is still widely deployed in production systems, such as several Google systems, including the Chubby Lock service [@burrows2006chubby] and the distributed NewSQL database system Spanner [@corbett2013spanner]. Most recent consensus protocols and most of the consensus literature are more or less based on Paxos or arose as a consequence of Paxos[^blockchain-exception].

[^blockchain-exception]: One exception is the blockchain, which uses distributed, decentralized consensus protocols, see subsection [@sec:blockchain-consensus].

\paragraph{History of Paxos.}

Paxos has been introduced by Leslie Lamport[^lamport-turing] in 1998 in the paper "The Part-Time Parliament" [@lamport1998paxos]. As allegorical as the title sounds, the paper also describes the Paxos algorithm in a picturesque approach. Just read the abstract to understand what we mean: "Recent archaeological discoveries on the island of Paxos reveal that the parliament functioned despite the peripatetic propensity of its part-time legislators. The legislators maintained consistent copies of the parliamentary record, despite their frequent forays from the chamber and the forgetfulness of their messengers. The Paxon parliament's protocol provides a new way of implementing the state machine approach to the design of distributed systems." The remainder of the paper is equally entertaining to read, but at the same time reveals a groundbreaking discovery in distributed consensus, but lacks enough detail to implement it. Due to the prevalence of Paxos, Lamport has attempted to make the algorithm more understandable and complete in his subsequent work [@lamport2001paxos], but it has been shown that people of all types (students as well as engineers) still have difficulty fully understanding it [@ongaro2013raft].

[^lamport-turing]: Leslie Lamport received the Turing Award in 2013 for all of his "fundamental contributions to the theory and practice of distributed and concurrent systems, notably the invention of concepts such as causality and logical clocks, safety and liveness, replicated state machines, and sequential consistency" [@lamport2013turing]. Among other things, he authored, discovered and invented the LaTeX typesetting system, the $\textrm{TLA}^{+}$ formal specification language, Paxos, State Machine Replication and the Byzantine Agreement problem.

Further improvements and alterations on Paxos where made in the past, some by Lamport himself such as Fast Paxos [@lamport2006fast], which introduces active replication characteristics (cf. subsection [@sec:active-replication]), and it has also been introduced for full transaction agreement in addition to a single value only [@lamport2006consensus].

Next to the Paxos consensus algorithm, there is the _Paxos algorithm_, which uses instances of the Paxos consensus algorithm in sequence to achieve fault-tolerant state machine replication. To reduce confusion, the first is referred to as _Basic Paxos_, while the latter is called _Multi-Paxos_.

\paragraph{Formal Verification.}

The Paxos consensus algorithm has been formally verified by Lamport using the $\textrm{TLA}^{+}$ formal specification language (see subsection [@sec:cost-of-replication] for reference)[@lamport2006fast]; Multi-Paxos has also been formally verified by Chand et al. [@chand2016formal].

\paragraph{Consistency in Paxos.}

As for most consensus protocols, Paxos provides strong consistency through linearizability. Multi-Paxos uses the state machine replication approach, while the protocol does not put much emphasis on the state machine. Multi-Paxos executes the same set of commands in the same order, on multiple participants. Paxos fullfills the liveness and safety requirements of consensus protocols (see subsection [@sec:consensus-protocols]).

\breakedparagraph{Node Roles}

\noindent In Paxos, nodes play different roles during the execution of the algorithm: 

- **Proposers**: A node that tries to satisfy a client request by attempting to convince the acceptors to agree to its requests. In other protocols, they are often referred to as leaders or coordinators. Note that in Paxos, there can be multiple proposers. It is up to the implementation if there should be a single proposer only, or to have multiple (e.g., to support concurrency), which also introduces potentials for conflicts, which are handled by the Paxos algorithm.\vspace{4pt}
- **Acceptors**: Nodes that listen and response to requests from proposers, able to form a quorum. Each message sent to an acceptor must be sent to and accepted by a quorum of acceptors.\vspace{4pt}
- **Learners**: Additional nodes in the system which just "learn" the final values that are decided upon. They do not participate in a quorum of acceptors, but can still accept values that are replicated. Learners allows to reduce the read latency and availability of the system (i.e. in geo-replication), while they do not contribute to the safety, crash-fault tolerance and write availability of the system. Similar to a proposer once a learner receives a majority of accept messages then it knows that consensus has been reached on the value.

In practice, one node can have two or all these roles at the same time. From the point of view of the protocol, these roles are considered to be single, seperate virtual instances. For example, you could run a setup of 5 nodes, where 3 nodes act as acceptors (so a quorum of 2 can be formed), one of these nodes act as a proposer, and all the 5 nodes act as learners (therefore executing the actual operations after the agreement). The 2 nodes that do not act as acceptors do not contribute to the safety and fault-tolerance of the system, but this helps to improve the read latency of the system.

\begin{figure}[h!]
  \centering
  \includegraphics[width=0.7\textwidth]{images/paxos-setup.pdf}
  \caption[Common Paxos setup]{A common Paxos setup. Note that these are not neccessarily 6 different nodes, as one node can incorporate multiple roles at the same time.}
  \label{fig:paxos-setup}
\end{figure}

\breakedparagraph{Protocol Phases}

\noindent Due to the allegorical description of Paxos in the work of Lamport, there are multiple slightly different descriptions and implementations of the actual Paxos algorithm. Although they differ in detail, their outcome is the same. We present one of those.

Paxos works in two phases to make sure multiple nodes agree on the same value in spite of partial network or node failures. The phases are illustrated in figure \ref{fig:paxos}. These phases can go in multiple rounds per Paxos instance if some failure happen (such as network partitioning, node crashes, message timeouts or conflicting proposers, to mention a few). In Basic Paxos, a Paxos instance ends once the client request is satisfied.

\paragraph{Phase 1a: Prepare.}

Once a client requested a write, a proposer accepts the request. It then chooses a new proposal version number $n$ (a generation clock, see subsection [@sec:state-machine-replication]) and sends a $\mathtt{prepare}(n)$ request to all the acceptors.

\paragraph{Phase 1b: Promise.}

Acceptors receiving this request will compare that with the last proposal number it knows:

- If the received $n$ is greater than any proposal number of a request they had already responded to, the acceptors responses with a promise. This promise comes with the guarantee that this acceptor will not accept any other proposals numbered less than $n$ from now on. If a acceptor recently accepted a previous proposal during this paxos instance, it sends the previous proposal number $n'$ and value $v'$ of the latest accepted proposal in $\mathtt{promise}(n, n', v')$, otherwhie it will just send $\mathtt{promise}(n)$.
- Else, the acceptor rejects the request, as it has already seen a higher proposal number, and sends a $\mathtt{nack}$ response to notify the proposer that it can stop the round.

\paragraph{Phase 2a: Accept.}

If the proposer receives promises from a quorum of the acceptors, then it will issue a request to accept a value $\mathtt{requestAccept}(n, v)$ with the proposal number $n$ and the value to write $v$. The value to write is either 
- the value $v'$ associated with the highest proposal number $n'$ if any sent by the acceptors (to satisfy the consistency property that "at most one value can be learned" [@lamport2006fast]),
- else the original value it wanted to propose.

If the proposer does not get enough promises to reach a quorum in a certain timeframe, it restarts the whole process.

\paragraph{Phase 2b: Accepted.}

If an acceptor receives an accept request, it accepts the proposal unless it has already responded to a prepare request with a number greater than $n$. Whenever an acceptor accepts a proposal, it responds to the proposer and all the learners with $\mathtt{accepted}(n, v)$. Once the proposer receives $\mathtt{accepted}(n, v)$ from a quorum, it decides on the value $v$, writes it to its state and replies to the client. Similarly, once a learner receives the command from a quorum, it also decides on this value.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/paxos.pdf}
  \caption[The Paxos consensus algorithm]{Idealistic message sequence in the Paxos consensus algorithm}
  \label{fig:paxos}
\end{figure}

There are also variants where there is an explicit third learning phase where not the acceptors talk to the learners, but the proposer sends a single $\mathtt{decide}(v)$ command to them after it decided on the value based on the quorum. This is illustrated in figure \ref{fig:paxos-learn-phase}. Another variant requires the learners to reply to the client, not the proposer. By setting one node to be both a proposer and a learner, the same behavior as in the presented approach can be achievend. All thse variants have slightly different consequences, especially when it comes to network partitioning. 

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/paxos-learn-phase.pdf}
  \caption[Alternative learn phase in Paxos]{Alternative learn phase in the Paxos consensus algorithm}
  \label{fig:paxos-learn-phase}
\end{figure}

paragraph{Multi-Paxos.}

Multi-Paxos[^no-clear-multi-paxos] allows for log replication, which leads to state machine replication, by allowing to agree on a sequence of values rather than just a single value. It runs seperate instances of Basic Paxos for each entry in the log. Additional to the proposal number, it comes with a monotonically increasing index number which is added to Prepare and Accept phase requests. Each single Paxos consensus instance can commit their values out-of-order, which makes the Multi-Paxos log working as follows: if a log is decided to a certain index, then all subsequently decisions will extend this log (i.e. the previously decided log up to the high-water mark is a common prefix). It is not guaranteed that the logs after the high-water mark are consistent in a quorum, though. They may have holes or even differing values at the same index, as long as they haven't been decided upon, as sketched in figure \ref{fig:multi-paxos-logs}. This is an important characteristic: the state machine should only be served with the operations in the log up to the high-water mark, and client requests only replied to successfully once the log entry and all all previous log entries are decided. After the mark, the logs may only eventually converge. Not all acceptors logs need to know the whole log up to the high-water mark, as only a quorum is needed. The learners are responsible to provide the consistent log up to the high-water mark to the state machines.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.75\textwidth]{images/multi-paxos-logs.pdf}
  \caption[Logs in Multi-Paxos]{Logs in Multi-Paxos may have holes and differ after the high-water mark. Only a quorum of correct log entries is needed to achieve consistency. At each index, the logs are viewed seperately.}
  \label{fig:multi-paxos-logs}
\end{figure}

[^no-clear-multi-paxos]: Readers trying to find one proper specification of Multi-Paxos in acedemic literature will fail. It was not specified in detail in literature; the modern understanding of Multi-Paxos is derived from practical implementations in popular distributed systems (of vendors such as Google or Microsoft) and there are also differences in these implementations.

In Multi-Paxos, phase 1 can skipped for subsequent requests as long as the cluster stays healthy, since the quorum already decided to accept values from one proposer. This proposer can be decided to be a single leader, which reduces potential proposer conflicts. Phase 1 will be reintroduced once there is a conflict, which is detected in phase 2.

Why are there two phases at all? As long as the whole cluster stays healthy, the first phase could be omitted. But in case of a node crash, subsequent requests need to know what happened previously and if it was fully committed. Phase 1 is therefore needed to determine whether anything remains to be done. Phase 2 is then used to set the correct value to ensure consistency of all processes within the system. Due to the complexity of Paxos, this can be best understood by examples. We recommend that readers look for Paxos examples of failures to understand the algorithm and why it provides fault-tolerance.

#### ZooKeeper Atomic Broadcast (ZAB) {#sec:zookeeper}

Introduced by Yahoo!, later became an Apache project. 

"ZooKeeper is a highly-available
coordination service used in production Web systems such as
the Yahoo! crawler for over three years."

It combines atomic broadcast (which is similar to consensus, but generalized for all types of message broadcasts in distributed system; consensus for crash-faults can be reduced to atomic broadcast and vice-versa, see TODO) and primary-copy (cf. subsection [@sec:primary-copy]) with leader-election

[@junqueira2011zab]

ZooKeeper itself is linearizable [@hunt2010zookeeper], while ZAB as a primary-copy...

". ZooKeeper implements a primary-backup scheme in which a primary process executes clients operations and uses Zab to propagate the corresponding incremental state changes to backup processes.  Zab must guarantee
that if it delivers a given state change, then all other changes it
depends upon must be delivered first."

With ZooKeeper, a primary process receives all
incoming client requests, executes them, and propagates the
resulting non-commutative, incremental state changes in the
form of transactions to the backup replicas using Zab, the
ZooKeeper atomic broadcast protocol.

Critical to the design of Zab is the observation that each
state change is incremental with respect to the previous state,
so there is an implicit dependence on the order of the state
changes. State changes consequently cannot be applied in any
arbitrary order, and it is critical to guarantee that a prefix of the
state changes generated by a given primary are delivered and
applied to the service state. State changes are idempotent and
applying the same state change multiple times does not lead to
inconsistencies as long as the application order is consistent
with the delivery order.


ZooKeeper itself is an additional service that must be deployed as an external agent next to the actual replicated or coordinated system, while ZooKeeper itself replicates its information with ZAB. This approach comes with disadvantages: it adds additional maintenance efforts (a second service must be maintained, running on a cluster of seperate (virtual) nodes), reduces the overall availability since a system using ZooKeeper relies on it and becomes unavailable as soon as ZooKeeper is unavailable (see subsection [@safety-reliability] for reference), or during a network partitioning, and reduces the overall latency of the replicated system, as coordination with the ZooKeeper cluster adds an additional step.

#### Viewstamped Replication

- Based on Primary-Copy https://pmg.csail.mit.edu/vr/oki88vr-abstract.html and State machine replication
- offers a reconfiguration protocol
- handles fail-stop but not Byzantine failures

ABC [@oki1988viewstamped], [@liskov2012viewstamped]

#### Leaderless Byzantine State Machine Replication {#sec:leaderless}
SMR in general means that there is a single leader (as in Paxos and Raft...)...


Geo-replication... Due to their reliance on a leader replica, classical SMR protocols offer
limited scalability and availability under a geo-replicated setting. To solve this problem, recent protocols follow instead a leaderless approach, in which each replica is able
to make progress using a quorum of its peers. These new leaderless protocols
are complex and each one presents an ad-hoc approach to leaderlessness.

there are multi-leader approaches (TODO reference multi-leader raft) and most recent literature deals with leaderless SMR like Tempo [@enes2021tempo] or Wintermute [@rezende2021leaderless]...

Leaderless SMR shows its strengths especially when used in blockchains... high cost of replication... especially when strong consistent... see [Blockchain Consensus Protocols](#sec:blockchain-consensus)

#### Blockchain Consensus Protocols {#sec:blockchain-consensus}

In recent years, the problem of byzantine fault tolerant consensus has raised significantly more attention due to the widespread success of blockchains and blockchain-based applications, especially cryptocurrencies such as Bitcoin, which successfully solved the problem in a public setting without a central authority...

Blockchains are distributed systems par excellence... Using strong consistent consensus protocols to ensure every node reads the same... Fail-stop is not a problem due to the large amounts of nodes in such a network, but byzantine fault-tolerance is crucial to those consensus protocols to secure a blockchain.

Blockchain protocols generally belong to the class of BFT consensus algorithms, as crashed nodes are not a problem in general, but faulty values are. We describe this briefly in subsection [@sec:blockchain-consensus].

As shown in subsections [@calm] and [@consistency-decisions], monotonicly growing append-only systems can be replicated with eventual consistency in a coordination-free way. Blockchains are distributed ledgers that provide this monotonic characteristics... (do they all?) So, depending on the requirements of the chain, a great range of consistency levels are possible: strong consistency with coordination between nodes to agree not only on a final state, but also to output the same order of blocks and to only expose the ledger state when all are ready to expose... or eventual consistency are possible (TODO show examples for both)

<!-- 
 A blockchain [40] is an append-only, tamper-resistant distributed ledger of transactions organised in a chain of blocks, originally used for cryptocurrencies

 Ethereum [49] popularised the concept of smart contracts as a mean to implement any
application on top of a blockchain. These first proposals, however,
have been criticised because the underlying blockchain protocol
used to update the ledger, based on proof-of-work, is extremely
energy consuming. New proposals then subsequently emerged
proposing alternatives to proof-of-work, including a notable number using standard Byzantine fault-tolerant distributed consensus
(BFT consensus for short)
-->

"Within a Blockchain network system, the
strong consistency model means that all nodes have the same ledger at the same time, and during the time when the
distributed ledger is being updated with new data, any subsequent read/write requests will have to wait until the
commit of this update"

" The consistency property has raised some controversial debate. Some argue
that Bitcoin systems only provide eventual consistency [6], which is a weak consistency. Others claim that Bitcoin guarantees strong consistency, not eventual consistency [7]." [@kanga2020management]

BTC seeems to be sequentially consistent, as all nodes must agree on the same sequence, but only in program order, and not all must see transactions at the same time, and the consistency is controlled with the mining difficulty in the PoW consensus...

"certain Blockchain applications are less risk-averse and may benefit from a weaker consistency guarantee for convenience and performance. "

https://media.springernature.com/lw685/springer-static/image/art%3A10.1007%2Fs11276-019-02195-0/MediaObjects/11276_2019_2195_Fig1_HTML.png

<!-- From https://tel.archives-ouvertes.fr/tel-03584254/document p24:

In more detail the Bitcoin protocol works as follows: Processes communicate
via reliable FIFO authenticated channels (implemented with TCP), thus modeling
a partially synchronous system [58]. When a transaction is sent by a client it
is placed in a shared pool, processes called miners collectively run a repeated
consensus protocol to select which transaction will be appended to the ledger.
In this consensus protocol, miners solve a cryptographic puzzle (proof-of-work)
to produce a valid block of transactions (i.e. miners are basically going through
a voting process, where they vote with their CPU power for valid blocks). This
proof-of-work procedure is computationally expensive (therefore, economically
expensive as well) to curb the effectiveness of Sybil attacks.
The advantage of using this approach is that it scales well with the numbers
of miners if one consider the system’s safety, since the more miners there are, the
more secure the service becomes (i.e. it gets increasingly more expensive to do
attacks). The downside is that Nakamoto’s consensus it is not strictly speaking
consensus. It can be the case that two miners concurrently solve the puzzle and
then append blocks to the blockchain in a way that neither block precedes the
other. This is called in the Bitcoin community as a fork. Typically, processes
continue to build on the block with the longest chain, that is, the one which has
more work done.
Dealing with forks highlight the disadvantages of the protocol, for instance: it
takes a waiting time of about 10 minutes to grow the chain by one block and from
an application point of view, waiting a few blocks is required (6 are recommended
in Bitcoin [26]) to guarantee that the transaction remains in the authoritative
chain. Therefore, the possibility of forks affects the consistency guarantees and
throughput of transactions (about 3 to 10 transactions per second in Bitcoin).
Many papers [11, 66, 110, 73] have been published with the intent of formalizing PoW protocols like Bitcoin and studying its consistency guarantees. In [66]
the authors provide one of the first formalizations of the Bitcoin protocol and
show that under synchronous environment assumptions the protocol guarantees
whp. an eventual consistent prefix. More broadly, in [11] the authors introduce the notion of Eventual Prefix and show that Bitcoin and other permissionless protocols abide to it. In [73, 72] the authors argue that Bitcoin implements
Monotonic Prefix Consistency whp.

-->

PoW (like the Nakamoto Consensus Algorithm that powers Bitcoin [@nakamoto2008bitcoin]), PoS, PoX, PBFT (see previous paragraph and [@li2020scalable])... all are BFT if conditions apply (especially, no 51% attack - remember, you need $2k + 1$ nodes to tolerate $k$ byzantine faulty nodes) (most important to prevent those failures!), BFT State Machine Replication protocols..

As blockchains typically have hundreds to hundred of thousands of replicas and need to provide high levels of consistency, classic single-leader state machine protocols are not applicable here, as they will dramatically slow down the whole chain due to their cost of replication.



TODO this is nice to have, remove if not enough time to mention this. If yes, reference (https://arxiv.org/pdf/1810.03357.pdf and https://arxiv.org/pdf/1904.04098.pdf)

### Summary

In this chapter, we have shown that replication always involves tradeoffs. There are several theorems that state that you cannot have consistency at the same time with full availability and low latency. Several important consistency models have been discussed, including their tradeoffs and limitations. It was shown that the limitations strongly depend on the use cases of the replicated data store. Finally, relevant replication protocols were discussed with respect to these consistency models, both from a theoretical and practical point of view. We presented protocols that are heavily used in production systems such as Paxos, CRDTs, blockchain consensus protocols or Viewstamped Replication, as well as historic protocols such as ROWA and primary-copy that are only in expectional use as of today.

In the next chapter, we will take a closer look at one specific protocol for replicating state machines with strong consistency that yields interesting properties for our work: the Raft consensus protocol.

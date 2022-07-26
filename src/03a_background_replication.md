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

\todo{Verify phrasing of this subsection}

Standalone applications can benefit from _scaling up_ the hardware and leveraging concurrency techniques/multithreading behavior (in this case we speak of _vertical scalability_), but only to a certain physical hardware limit (such as CPU cores, network cards or shared memory). Approaching this limit, these applications and services can no longer be scaled economically or at all, as the hardware costs increase dramatically. In addition, applications running on only one single node pose the risk of a single point of failure, which we want to counter with replication.

To scale beyond this limit, a system must be designed to be _horizontally scalable_, that is, to be distributable across multiple computing nodes/servers (also known as to _scale out_). Ideally, the amount of nodes scales with the amount of users, but to achieve this, certain decisions must be made regarding [consistency models](#sec:consistency) and [partitioning](#sec:partitioning). Legacy vertically scalable applications cannot be made horizontally scalable without rethinking the system design, which includes replication and message passing protocols between the nodes to which the application data is distributed.

With vertical scaling, when you host a data store on a single server and it becomes too large to be handled efficiently, you identify the hardware bottlenecks and upgrade. With horizontal scaling instead, a new node is added and the data is partitioned and split between the old and new nodes.

\todo{Illustration of vert. vs horz. scalaing}
TODO this chart https://www.cloudzero.com/hubfs/blog/horizontal-vs-vertical-scaling.webp

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
  \includegraphics[width=1.2\textwidth]{images/mtbf_relation.pdf}
  \caption{Relation of the Mean Time Between Failure (MTBF) and other reliability metrics}
  \label{fig:mtbf-relation}
\end{figure}

Actually, the MTBF can also be expressed as an integral over the reliability function $R(t)$ [@birolini2013reliability], which illustrates the relation between the two metrics:

$$ \textrm{MTBF} = \int_{0}^{\infty} R(t) dt $$

\begin{figure}[h]
  \centering
  \includegraphics[width=0.6\textwidth]{images/mtbf-r-curve.pdf}
  \caption{A curve for a typical $R$ function, illustrating the relation between the $R$ and MTBF metric}
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

#### Types of Possible Faults {#sec:possible-faults}

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
  \includegraphics[width=1.2\textwidth]{images/error-classes-venn.pdf}
  \caption{Relation of the different fault models}
  \label{fig:error-classes-venn}
\end{figure}

#### Partition-Tolerance

In addition to the discussed types of faults, there is also the risk of _network partitioning_. There are various types of network partitions, including complete, partial and simplex partitions. In figure \ref{fig:error-classes-venn}, all three are shown for reference. In (a), we have a complete partitioning of the network, resulting in a split into two disconnected groups, also known as a _split brain_. In (b), not all nodes are affected by the partitioning, so there is at least one route between all nodes, which is called _partial partition_. In (c), there is a _simplex partition_, in which messages can only flow in one direction. Network partitioning events can be of a temporary or persistent nature, depending on the original fault that caused them, but also on the strategies used to resolve them. In large-scale distributed systems, network partitioning is to be expected. Particularly in massive-scale distributed systems like blockchains, network partitioning is part of the design.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/partitioning-all.pdf}
  \caption{Possible network partitioning types. (a) Complete partition/split brain, (b) partial partition, (c) simplex partition.}
  \label{fig:partitioning-types}
\end{figure}

If not handled correctly, the partitioning can lead to Byzantine behavior once the partitioning is resolved again. Partitioning also compromises consistency, as the resulting subnetworks may serve different clients, aggregating different sets of data until they are reunited. Alquraan et al. found that network-partitioning faults lead to silent catastrophic failures, such as data loss, data corruption, data unavailability, and broken locks, while in 21 % of failures, the system remains in a persistent faulty state that persists even after partitioning is resolved [@alquraan2018analysis]. Those faults occur easily and frequently: in 2016, a partitioning happened once every two weeks at Google [@govindan2016evolve] and in 2011, 70% of the downtime of Microsoft's data centers where caused by network partitioning [@gill2011understanding]. Even partial partioning causes a large number of failures (due to bad system design), which may be suprising as there is still a functioning route for messages to pass through the network.

Partition-tolerance is in general a must-have for a distributed system. In the case of partitioning, users and clients expect the system to still be available and reliable, without compromising the safety by subsequent faulty behavior. The don't want to experience the interruption by the partitioning at all. For a replication protocol to be truly fault-tolerant for large-scale or geographically distributed systems, partitioning must be considered, as it is unavoidable, even partial partitioning as it has been shown. In consensus protocols, this is a mandatory requirement. Container orchestration services like Kubernetes help in resolving partitioning by _network reconfiguration_ if they detect node failures and partitioning, which must be taken into account be the consensus protocol.

#### Disaster Recovery

Another type of failure that is important to address are _disasters_ in the data center. _Disaster recovery_ includes all actions taken when a primary system fails in such a way that it cannot be restored for some time, including the recovery of data and services at a secondary, surviving site. One important disaster recovery measure is _geo-replication_, which can be used to manage disasters of the type that render data centers unavailable due to, for example, a natural disaster such as a flood or earthquake. Geo-replication is discussed in more detail in subsection [@sec:geo-replication]. Since disaster recovery is a complex subject area of its own that goes beyond the application of replication mechanisms, it is not discussed in detail in this thesis (apart from geo-replication).

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
    \caption{Various availability classes and the respective annual downtime, based on the Availability Environment Classification (AEC) of the Harvard Research Group (HRG)}
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
  \caption{Example for the resulting availability of a sequential orchestration of services}
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
  \caption{The data-centric and client-centric perspective on consistency models, as described by Bermbach et al.}
  \label{fig:data-client-centric-perspective}
\end{figure}

Following, a few of those consistency models are discussed, starting with the model with the strongest consistency guarantees [@steen2007distributed]. Throughout the discussion, we use the following notation (also see figure \ref{fig:consistency-timeline-notation}) to illustrate the behavior of the models by examples:

- $w(x, v)$ denotes a successful write operation of the value $v$ into the variable $x$.
- $r(x) = v$ denotes a read of $x$ that returns the value $v$.

\begin{figure}[H]
  \centering
  \includegraphics[width=0.8\textwidth]{images/consistency-timeline-notation.pdf}
  \caption{Notation of an operation invocation to illustrate following consistency discussions}
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
  \includegraphics[width=1.5\textwidth]{images/strong-consistency-flow.pdf}
  \caption{Communication model between clients and replica cluster nodes to ensure strong consistency. A write is acknowledged only when it is consistent throughout all cluster nodes, therefore synchronizing the operations.}
  \label{fig:strong-consistency-flow}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-linearizable.pdf}
  \caption{Operation schedule that satisfies the realtime ordering guarantee of the strong consistency model (linearizability).}
  \label{fig:consistency-ordering-linearizable}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-linearizable-overlap.pdf}
  \caption{A strongly consistent operation schedule with a read-write overlap. During the overlap, the read is allowed to return either the current or the previous write.}
  \label{fig:consistency-ordering-linearizable-overlap}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-linearizable-write-overlap.pdf}
  \caption{This operation schedule is still linearizable, as the order of committing the overlapping writes is not guaranteed.}
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
  \includegraphics[width=1.5\textwidth]{images/sequential-consistency-flow.pdf}
  \caption{Schematic communication model between clients and replica cluster nodes for sequential consistency. A write can be acknowledged before it is propagated consistently across all cluster nodes. For all successful writes that are committed throughout the cluster, the original order is guaranteed.}
  \label{fig:sequential-consistency-flow}
\end{figure}

All reads at all nodes will see the same order of writes to ensure sequential consistency, but not necessarily in the absolute order (by timestamps) in which clients requested the reads, as the latter can often be impractical as it can lead to reordering of operations between nodes when concurrent writes appear in the wrong order. With sequential consistency, programmers must be careful because two successive writes from different nodes (in real time) to the same value can occur in any order, resulting in unexpected overwrites.

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-sequential.pdf}
  \caption{Operation schedule that satisfies the global ordering guarantee of the sequential consistency model. The writes of $\textrm{N}_1$ and $\textrm{N}_2$ are seen by $\textrm{N}_3$ in the order of their program execution on the particular nodes, while the order of the interwoven operations does not neccessarily correspond to the real-time sequence. Both $\textrm{N}_3$ and $\textrm{N}_4$ have also agreed on one (partial) global order, while $\textrm{N}_4$ experiences updates faster.}
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
  \includegraphics[width=1.5\textwidth]{images/causal-consistency-flow.pdf}
  \caption{Schematic communication model between clients and replica cluster nodes for causal consistency. A write can be acknowledged before it is propagated across all cluster nodes. For all causally sequential writes that are committed throughout the cluster, the original order is guaranteed.}
  \label{fig:causal-consistency-flow}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-causal-violation.pdf}
  \caption{Operation schedule that violates causal consistency. As the writes of $\textrm{N}_1$ has already been observed by $\textrm{N}_2$ before overwriting it (assuming that its observation may have triggered the overwrite, thus they are causally related), it must be seen in this order by all subsequent reads of all nodes. Since $\textrm{N}_4$ reads the most recent (the overwriting) value before the overwritten one, it ignores the causal relation and thus violating the properties of causal consistency.}
  \label{fig:consistency-ordering-causal-violation}
\end{figure}

\begin{figure}[h]
  \centering
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-causal.pdf}
  \caption{Operation schedule that meets the requirements for causal consistency. Since there is no causal relation between the writes of $\textrm{N}_1$ and $\textrm{N}_2$, any order of occurrence of the values in subsequent reads satisfies the properties of causal consistency.}
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

\begin{figure}[h]
  \centering
  \includegraphics[width=1.5\textwidth]{images/eventual-consistency-flow.pdf}
  \caption{Schematic communication model between clients and replica cluster nodes for eventual consistency. A write is acknowledged immediately before it is propagated across all cluster nodes. The system remains available even when partitioned by allowing disconnected nodes to converge later when reconnected again.}
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
  \includegraphics[width=1.2\textwidth]{images/consistency-ordering-eventual.pdf}
  \caption{Operation schedule for eventual consistency. One node is partitioned from the others, returning an inconsistent, outdated state in between. When reconnected, the state of the partition converges and eventually becomes consistent again.}
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
    \caption{Consistency models in descending order of strictness and at the same time in ascending order of availability}
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
  \includegraphics[width=1.6\textwidth]{images/pacelc.pdf}
  \caption{Illustration of the PACELC theorem, showcasing the trade-offs in the case of partitioning and also in the absence of network partitioning}
  \label{fig:pacelc}
\end{figure}

The author introduced this theorem to support decision making in the design and implementation of a distributed database system, since consistency decisions must also be made outside of network partitions: the trade-off between consistency and latency is often even more important and arises as soon as a distributed database system introduces replication.

\todo{Further explanation if sufficient time}

#### The CALM Theorem {#sec:calm}

The CALM theorem (Consistency As Logical Monotonicity) arose from a critique of the CAP theorem. It helps to understand whether a distributed problem can be solved with a strong or weak consistency model. While a strong consistency model always requires some form of coordination between nodes which increases latency, a weak model such as eventual consistency can eschew coordination entirely, but often at the cost of violating correctness properties. The CALM theorem allows the system designer to decide whether a weak consistency model can be applied without compromising correctness by considering possible monotonic properties. The theorem says "a problem has a consistent, coordination-free distributed implementation if and only if it is monotonic [@hellerstein2020keeping]".

Monotonicity is defined as follows: A problem $P$ is monotonic if for any input sets $S$, $T$ where $S \subseteq T$, $P(S) \subseteq P(T)$. Figure [@fig:monotonic-function] illustrates this using a simple function example.

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

One way to achieve this behavior is to apply the _derived monotonic state pattern_ [@braun2022calm]. The pattern is illustrated in figure \ref{fig:derived-monotonic-state-aggregate}, as initially described by Braun. The pattern can be applied by factoring out every operation on the state object into _immutable aggregates_, such as _domain events_. We will illustrate this with the example of a shopping cart. Domain events in a shopping cart are the insert and removal of items, denoted as `itemInserted` and `itemRemoved`. In a naive implementation, both events could be modeled as operations on a mutable state (the shopping cart entity). But with a weak consistency model, the order of these operations can not be guaranteed, as illustrated in figure [@fig:shopping-cart-naive].

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/shopping-cart-naive.pdf}
  \caption{Naive implementation of a shopping cart with weak consistency. $N_2$ received the operations in a different order than $N_1$, rendering a wrong final state.}
  \label{fig:shopping-cart-naive}
\end{figure}

But this problem can be implemented in a monotonic fashion: instead of applying the domain events on a single mutable state (an _activity aggregate_), they could just be persisted as immutable aggregates in an append-only manner, resulting in two separate monotonically growing sets. Once the final state needs to be observed, it can be derived from the domain events by putting them together on demand[^state-management-libraries]. The resulting state then describes a derived aggregate, expressed by the count of `itemInserted` minus the count of `itemRemoved` for each particular item.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/shopping-cart-monotonic-aggregate.pdf}
  \caption{Monotonic implementation of the shopping cart problem. Both nodes will derive the same final state without the need for coordination.}
  \label{fig:shopping-cart-monotonic-aggregate}
\end{figure}

Unfortunately, this monotonic behavior is not applicable to all types of operations. There are events that are naturally causally dependent on previous events. In our shopping cart example, this could be a checkout[^event-sourcing]: neither the add nor delete operation commutes with
a final checkout operation. If a `checkout` operation message arrives at a node before some insertions of the `itemInserted` or `itemRemoved` events, those events will be lost.

[^state-management-libraries]: This is also how modern state management systems like MobX or Redux (that are heavily used in frontend development) do this to ease complex implementation of concurrency problems on single devices by applying _functional reactive programming_ paradigms.

[^event-sourcing]: In _event sourcing_, monotonic characteristics can be useful. However, to be able to do _time travel queries_, strong consistency and realtime properties are needed again to derive any intermediate state, at least for all causally related events.

The monotonic property of a problem means that the order of operations does not matter at all. Consequently, in the case of network partitions, both consistency and availability are possible in a monotonic problem,  since replicas will always converge to an identical state on all nodes when the partition heals, and this without the need for any conflict resolution or coordination mechanisms.

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/derived-monotonic-state-aggregate.pdf}
  \caption{Factoring out a nontrivial activity aggregate into immutable and derived aggregates to acquire a monotonic state aggregate allows for a weaker consistency model and therefore lower latency without actually compromising consistency, according to the CALM theorem.}
  \label{fig:derived-monotonic-state-aggregate}
\end{figure}

#### Deciding for Consistency {#sec:consistency-decisions}

\epigraph{The first principle of successful scalability is to batter the consistency mechanisms down to a minimum, move them off the critical path, hide them in a rarely visited corner of the system, and then make it as hard as possible for application developers to get permission to use them.}{--- \textup{James Hamilton, VP \& Distinguished Engineer at Amazon}}

In order to decide on a consistency model, several factors must be taken into account. The actual consistency model to decide for depends on the use case of the distributed database system. Therefore, many popular DDBS allow the developers to select the consistency model of their choice by configuration. When deciding for a consistency model, it makes sense to base the decision on the dependability properties that are neccessary for the given use case (cf. the bank transfer example in subsection [@sec:safety-reliability]).

There are even attempts to formally describe the consistency model needs at the level of operations: Gotsman et al. propose a proof rule for determining the consistency guarantees for different operations on a replicated database that are sufficient to meet given data integrity invariants [@gotsman2016cause].

\paragraph{Challenges of weaker consistency.}

When applying a weaker consistency model, especially eventual consistency (cf. section [@sec:optimistic-replication]), challenges arise on the consuming application side. While a strongly consistent distributed system is similar to a single-node system for the consumer and makes it easy for the developer to use because its API is clear, stale data and long-running asynchronous behavior must be handled appropriately when talking to an eventual consistent system, which makes it a completely different API. Consequently, eventual consistency is not suitable for all use cases. 

\paragraph{Multiple levels of consistency.}

The weaker consistency models generally violate crucial correctness properties compared to strong consistency. A compromise is to allow multiple levels of consistency to coexist in the database system, which can be achieved depending on the use case and constraints of the system. It is even possible to provide different consistency models per operation or class of data. An example is the combination of strong and causal consistency for applications with geographically replicated data in distributed data centers [@bravo2021unistore]. In general, a microservice-oriented architecture that wraps up multiple services into a single application is strongly advised, as each microservice can provide its own specific and well-reasoned consistency model suitable for its particular purpose.

\paragraph{Constantly review decisions.}

When stronger consistency models increase the latency of a system in an unacceptable way and there are no other ways to mitigate this, eventual consistency may be considered. It can dramatically increase the performance of a system, but it must fit the use cases of the system and its applications, and it means additional work for developers. At least, it is worth questioning again and again whether strong consistency is really mandatory: even popular systems like Kubernetes undergo this process, as current research seeks to apply eventual consistency to meet edge-computing requirements [@jeffery2021rearchitecting]. As the authors of the CALM theorem describe, instead of micro-optimizing strong consistency protocols, it is often more effective to question the overall solution design (i.e., perhaps a monotonic solution can be found) and minimize the use of such protocols (cf. subsection [@sec:calm]). At the same time, systems that have originally been eventually consistent could also benefit from re-architecting into stronger consistency to be easier to use and understand for both users and developers.

There are examples that anecdotally show that high availability and high throughput are possible even under strong consistency constraints: AWS S3 introduced strong consistency for their file operations[^s3-eventual] recently in 2021 [@amazon2021s3consistency]. They even claim to deliver strong consistency "without changes to performance or availability", compared to the earlier eventual consistency.

[^s3-eventual]: Bucket configuration operations are still eventual consistent in AWS S3 at the time of this writing.

In essence, it is worthwhile to constantly challenge applied consistency models as use cases change or new technologies and opportunities emerge.

\paragraph{Let the user decide.}

When deciding on a consistency model for a distributed database system, it is important to recognize that different applications built on top of the database will themselves have different consistency requirements, so it may be a good idea to provide multiple consistency models and flexibility in configuring these models for different types of operations in a database system. Many popular database system vendors allow their users to choose for the consistency model of their choice, even at the operations level. Some of those vendors offer true fine-grained consistency options, such as Microsoft Azure Cosmos DB, which offers even 5 models with gradually decreasing consistency constraints [@microsoft2022cosmosconsistency].

\todo{RabbitMQ consistency choices (quorum queues vs mirror queues) if time}

\paragraph{Immutability changes everything.}

Immutability naturally creates monotonicity, as the set of data—let it be either a payload or commands on this payload, like the `itemInserted`/`itemRemoved` example in subsection [@sec:calm] above—can only grow. Helland claims in his paper "Immutability Changes Everything" that "We need immutability to coordinate at a distance and we can afford immutability, as storage gets cheaper" [@helland2015immutability]. The latter statement is somewhat reminiscent of Moore's Law.

By designing a system to be append-only, and therefore monotonic, we receive all the benefits of coordination-free consistency. Append-only systems not only provide lower latency when replicated, but also better write performance at the local disk level. Many databases are equipped with a write-ahead transaction log that records all the changes to be made to the database in advance. These write-ahead logs allow for reliable and high-speed appends, because records are appended immutably, atomically, and sequentially. The log contains virtually the truth about the entire database and allows to validate past and recent transactions (e.g., in the event of a crash), as well as time travel to previous states of the database, acting like a ledger. Even redo and undo operations are stored in this log. As shown in subsection [@sec:calm], replicated and distributed file systems depend on immutability to eliminate anomalies. By deriving aggregates from append-only logs of observed facts, consistency can be guaranteed to a certain degree[^tampering-logs]. From a particular perspective, a database is nothing more than such a large, derivative aggregate. Append-only structures also increase the safety of a system and thus support its fault-tolerance: if a system is limited to functional computations on immutable facts, operations become idempotent. Then the system does not become faulty due to failure and restart.

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

\paragraph{Influence of the network infrastructure and overall architecture.}

Not only the use case, but also the capabilities of the infrastructure the system will be deployed onto (i.e., the network)  and the overall technical architecture play an important factor in the decision. Under certain circumstances, it is possible to provide high levels of consistency and yet low latency and availability, e.g., by using multiple or even nested layers of different consistency models and intelligent partitioning techniques.

The critique of the CAP theorem presented in the previous subsections allows for a more deliberate choice of consistency in practical systems, since several other properties can affect the actual requirements for consistency and dependability, often even more than the original theoretical properties of the CAP theorem. As an example, Google's distributed _NewSQL_ database system Spanner is in theory a CP class system [@corbett2013spanner]. It's design supports strong consistency with realtime clocks. In practice, however, things are different: given that the database is proprietary and runs on Google's own infrastructure, Google is in full control of every aspect of the system (including the clocks). The company can employ additional proactive strategies to mitigate network issues (such as predictive maintenance) and to reduce latency in Google's extremely widespread data center architecture. In addition, intelligent sharding techniques have been deployed (we will discuss partitioning and sharding in the following section [@sec:partitioning]) that take advantage of this data center architecture. As a result, the system is highly available in practice (records to date even show availability of more than five nines (99.999 %) at the time of writing), and manifested network partitions are extremely rare. Eric Brewer, the author of the CAP theorem and now (at the time of writing) VP of infrastracture at Google, even claims that Spanner is technically CP but effectively CA [@brewer2017spanner]. We'll look at Spanner in more detail in section [@sec:previous-work]. It is important to realize that this is difficult in practice for open, self-managed distributed databases, or generally for smaller, less complex infrastructures, or when there is no control over the underlying network, as this requires a joint design of distributed algorithms and new network functions and protocols (cf. the Eris protocol in subsection [@sec:coordination-free-replication]). And after all, it is always a question of overall economic efficiency.

In addition, the actual choice of a replication protocol adds another layer of considerations that must be factored into the decision. We will discuss the overall _cost of replication_ further in this work in subsection [@sec:cost-of-replication] once we have introduced additional concepts that play a crucial role in deciding on one or more consistency models and replication protocols.

As a rule of thumb, the more tightly coupled the replicated database system and infrastructure, the easier it is to ensure strong consistency without compromising latency and availability.

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
  \caption{a) Vertical vs b) horizontal partitioning, illustrated using a relational database table. In a), the table is split by attributes. In b), the table is split by records, using a lexicographical grouping.}
  \label{fig:partitioning}
\end{figure}

\paragraph{Vertical partitioning.}

In vertical partitioning, data collections[^data-collection] such as tables are partitioned by attributes. A collection is partitioned into multiple collections, each containing a subset of the attributes. An example of this are database systems where large data blobs that are rarely queried are stored separately (i.e., only when accessing the full set of details for a single record, but not when listing multiple records). Database normalization is also an example of vertical partitioning. Vertical partitioning comes in particularly handy when the partitioned data can be stored in separate file systems or hardware with different characteristics, depending on the read or write frequency and requirements of the different record contents. These requirements may also include dependability and consistency, so that, for example, less critical data may be given an eventual consistency model and lower availability guarantees. On the downside, vertical partitioning can make querying more complicated, so partitioning decisions must be made with proper justification and based on usage estimates.

The need for vertical partitioning can also be reduced by better upfront data design, such as splitting data in trivial facts and deriving an aggregated state instead of managing the a state by continously updating a single data record (cf. subsection [@sec:calm]). 

[^data-collection]: We will use the term _data collection_ to describe structured data as well as semi-structured data that belongs to a certain schema (that describes the structure of the data collection) in any kind of data stores: tables in relational databases, streams in event stores, topics in message brokers, or buckets in file storage systems, to mention a few.

In general, vertically partitioned data is not distributed across multiple nodes, as this would slow down queries: The chances that partitioned attributes of a single dataset will be requested in a single query are high. When partitions are distributed, partitioning strategies should be defined so that cross-partition queries are rare. For this reason, we will not discuss this issue further in this work.

\paragraph{Horizontal partitioning.}

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

\paragraph{Partitioning and replication.}

Neither replication or partitioning alone can make a system truly scalable and available. But if both techniques are applied, distributed databases can scale with their users, the data, and the operations executed on it, they can provide world-wide high availability, and they can even withstand data center outages. In common replicated and partioned architectures, the system is divided into _availability groups_. An availability group consists of a set of user databases that fail over together. It includes a single set of primary databases and multiple sets of replicas. Such groups are often deployed in multiple data centers in different geographic _availability zones_ to provide additional disaster resilience. Figure \ref{fig:load-balanced-sharding} illustrates this architecture. 

Load balancing becomes more challenging because replicas must be considered when distributing the load among the nodes. In large, horizontally scaled distributed systems, the _replication factor_ (the number of replicas per shard) is several magnitudes smaller than the number of total nodes. The load balancer (if it is a centralized agent) or the load balancing protocol (if the load balancer itself is decentralized) must maintain a distribution of replicas that balances the load on the nodes, which means that replicas can be moved between nodes during rebalancing.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/load-balanced-sharding.pdf}
  \caption{Simplified representation of a common replication and partitioning scheme for high availability (with a replication factor of 3). Shards are placed in availability groups (data centers) that are close to their most frequent writers. They are replicated across availability groups so that they can be read with low latency and remain available even in the event of a data center outage.}
  \label{fig:load-balanced-sharding}
\end{figure}

\todo{Work finished up to this point.}

<!-- now describe the difficulties and costs -->
\paragraph{Transactions on partitioned and replicated databases.}

Not only the replicated data within a partition, but also the transactions across partitions must follow a consistency model that meets the requirements of the system. In addition to replication between replicas of a shard, transactions must also be ordered and the atomicity of each transaction must be guaranteed. In general, transactions should be linearizable. Following the atomicity property of the ACID schema, every transaction should be applied to all shards it affects, or none at all.

Providing consistency and atomicity in the execution of transactions in a partitioned and replicated database system is a substantially greater challenge than simply arranging operations in a single replica group, since servers in different shards do not see the same set of operations but must still ensure that they execute cross-shard transactions in a consistent order.

Existing systems generally achieve this using a multi-layer
approach, as illustrated in figure [@fig:standard-partitioned-architecture]. A replication protocol is used to provide fault-tolerance and dependability inside of a replica group of a shard. Across shards, an commitment protocol provides atomicity, such as the _two-phase commit_ (2PC[^2PC]), which is combined with a concurrency control protocol for isolation, e.g. two-phase locking [@li2017eris]. With geo-replication, even another layer can be added on top to mirror the whole system into multiple geographical regions. Each of this layers adds its own coordination overhead, increasing the number of network round trips significantly and therefore the resulting latency, as illustrated in figure [@fig:2pc-plus-consensus]. NoSQL database systems therefore often forgo transactions altogether to improve availability and latency (see BASE properties of eventual consistent systems in subsection [@sec:consistency]), while NewSQL database systems again allow for transactions and provide ACID properties—mitigating the coordination problem through coordination-free replication and transactions across shards, as shown in subsection [@sec:coordination-free-replication].

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
  \caption{Coordination between shards and replicas for consistent transactions with traditional two-phase commits (2PC) and replication}
  \label{fig:2pc-plus-consensus}
\end{figure}

### Geo-Replication {#sec:geo-replication}

The replication techniques discussed so far in this work describe the replication of data within a single cluster to improve the dependability of the system. To improve performance when accessing the system and data from different geographical regions and to hedge against geographical limited disasters, data can be further replicated across clusters using _data mirroring_ techniques, which is referenced as _cross-cluster replication_.

TODO causal consistency! https://www.cs.uic.edu/~ajayk/ext/FGCS2018.pdf

TODO more from https://kafka.apache.org/documentation/#georeplication
- Also for feeding edge cluster data into one central cloud cluster
- In general, this is solved by simply cloning the full data in the background, either hot (while the primary cluster is still running) or cold (when the cluster is shutdown, to produce a reliable cold backup)

"Consistent
operations that span a wide area have a significant minimum round trip time, which can be tens of
milliseconds or more across continents. (A distance of 1000 miles is about 5 million feet, so at ½ foot
per nanosecond, the minimum would be 10 ms.) "

For geo-replication, a client-centric consistency model is sufficient as not all data 

TODO use and describe the term "availability zones". Maybe draw a diagram: Cluster -> Data Center -> Avail. Zone -> Countries + Continents

Strong consistency is in general a bad choice for geo-replication, as it requires writes to be committed to at least a global majority, reducing latency and availability dramatically. From a use case point of view, strong consistency is oftentimes even obsolete: The geo-replicas accessed by clients need only a fraction of the data of the other replicas most of the times (?). To prepare for disaster recovery, a trade-off between consistency and latency in the non-disaster case needs to be made, and is made on economical risk calculations: what's the cost if stale data is lost forever in the rare case of a disaster? "Within a globally distributed database environment there is a direct relationship between the consistency level and data durability in the presence of a region-wide outage. As you develop your business continuity plan, you need to understand the maximum period of recent data updates the application can tolerate losing when recovering after a disruptive event. The time period of updates that you might afford to lose is known as recovery point objective (RPO)." RPO: "the amount of data that can be lost within a relevant period of time for a company before significant damage occurs"
Note that use cases exists that actually require strong consistency on a global scale, but they are reduced on a very small subset of the data of the overall system, mostly metadata neccessary to manage and observe the whole system. In this cases, the RPO is 0, which requires strong consistency across geo-replicas

TODO multilevel possible: other consistent choices inside of a cluster then across clusters, and also dependent on the data/content.



When concerning about service latency, it is always desirable to place service replicas in close proximity to its clients. This allows reducing the request round-trip time and hence the latency experienced by clients

Providing lower latencies to users by serving data from nearby nodes...

\todo{Rephrase}
The replication performed by modern Internet services spans across several
geographical locations (geo-replication). This allows for increased availability
and low latency, since clients can communicate with the closest geo-graphical
replica. 


### Edge Computing

\todo{Very brief explanation (copy from later sections)}

- Strong concistency can be a problem
- Good system design can help (see next subsection) or using another consistency layer [@jeffery2021rearchitecting]

### Cost of Replication {#sec:cost-of-replication}

As far as we now discussed the benefits of replication for failure management, for Performance enhancements by reducing latency in edge computing and geographically distributed networks, and to increase dependability in general, we need to discuss the _Cost of Replication_. Dependent on the level of fault-tolerance, the consistency decisions and the selected protocol itself, the performance, especially for write operations, can decrease dramatically. We briefly discuss appropriate strategies for a performance-trade-off mitigation in this case.

TODO show the maths: how replication puts a upper bound on throughput

Modification on one replica triggers modification on all other replicas --> messaging and acknowledgment of all operations between replicas degrades performane

(So far, we discussed consistency decisions, partial repl., partitioning, geo-repl., multi-layer etc., so we are able to discuss the overall cost and trade-offs and make an educated decision)

- As mentioned, strong concistency can be a problem for performance as with higher cluster sizes, offering higher availability, write request latency
significantly increases and throughput decreases similarly (TODO refer to evaluation results)
- Many possible solutions:
  - Weaker consistency
  - Partial replication
    - Only metadata (TODO refer to InfluxDB)
    - Other/causal+ (see next paragraph)
  - Sharding + partitioning
    - Leveraging time splits
  - Multi-layer design (full replicated in the cloud, standalone on the edge...)

"With horizontal scaling, it is possible to keep a constantly high throughput rates (events/s). With higher replication factor, it decreases (TODO how does Zeebe keep throughput constant when latency increases???). Thanks to the buffer, intermediate higher event emitting rates can be compensated, if the mean rate stays below the max throughput... Otherwhise, the overall system slows down and probably crashes ATM (future work: Monitoring of the system, truely elastic, but due to append-only and linearizability we can not mitigate everything with partitioning if write rates stay too high for a minimum replica set; there will be a upper bound. Same would apply to eventual consistency btw: If mean write rate stays higher than max throughput, the system will neve become consistent (i.e., replica states will never converge (TODO should we mention that in fundamentals/cost of replication?)))"

#### Partial Replication

Similar to the considerations of client-centric consistency models... in many systems, not all clients need to access the same data. If data portions can be identified that are only accesses by some clients, they don't need to be consistently shared between all replicas...

TODO [@shen2015causal] (causal consistency for partial geo-replication)

###  Error Correcting Codes

TODO only a short excursion

### Replication Protocols {#sec:replication-protocols}

The next subsections describe the different categories of replication protocols and follow with a discussion of specific protocols.

#### Consensus Protocols {#sec:consensus-protocols}

\todo{Rephrase}

\todo{Talk about processes or nodes?}

TODO first mentioned by  (Thomas, Gifford, 1979)? Quorum

<!--
TODO The first studies on consensus protocols happened in the field of distributed, asynchronous systems of processes, but is also applicable to large-scale distributed systems of today
-->

"Consensus is a fundamental problem in fault-tolerant systems: how can servers reach agreement on shared state, even in the face of failures? This problem arises in a wide variety of systems that need to provide high levels of availability and cannot compromise on consistency; thus, consensus is used in virtually all consistent large-scale storage systems."

<!--
Raft diss:
Consensus algorithms for practical systems typically have the following properties:
• They ensure safety (never returning an incorrect result) under all non-Byzantine conditions,
including network delays, partitions, and packet loss, duplication, and reordering.
• They are fully functional (available) as long as any majority of the servers are operational
and can communicate with each other and with clients. Thus, a typical cluster of five servers
can tolerate the failure of any two servers. Servers are assumed to fail by stopping; they may
later recover from state on stable storage and rejoin the cluster.
• They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme
message delays can, at worst, cause availability problems. That is, they maintain safety under
an asynchronous model [71], in which messages and processors proceed at arbitrary speeds.
• In the common case, a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls; a minority of slow servers need not
impact overall system performance.
-->

"A consensus protocol enables a system of n asynchronous processes, some of which are faulty, to reach agreement." [@bracha1983resilient]

"As already defined in ... A protocol is k-fault tolerant if in the presence of up to k faulty processes it reaches agreement with probability 1."

"Each process starts with some initial value. At the conclusion of the protocol all the working nodes must agree on the same value. "

- achieve overall system reliability in the presence of a number of faulty processes
- coordinating processes to reach consensus
  - agree on some data value that is needed during computation

\todo{Rephrase}

Certain different variations of how to understand a consensus protocol appear in the literature. They differ in the assumed properties of the messaging system, in the type of errors allowed to the processes, and in the notion of what a solution is. Most notable is the differentiation between _consensus protocols for asynchronous and synchronous message-passing systems_. Fischer et al. have proven in the famous _FLP impossibility result_ (called after their authors) that a deterministic consensus algorithm for achieving consensus in a fully asynchronous distributed system is impossible if even a single node crashes in a fail-stop manner [@fischer1985impossibility]. They investigated deterministic protocols that always terminate within a finite number of steps and showed that "every protocol for this problem has the possibility of nontermination, even with only one faulty process." This is based on the failure detection problem discussed earlier in subsection [@sec:possible-faults]: in such a system, a crashed process cannot be distinguished from a very slow one.

As pessimistic as this may sound, it can easily be mitigated by adding nondeterminism to a protocol. Bracha et al. consider protocols that may never terminate, "but this would occur with probability 0, and the expected termination time is finite. Therefore, they terminate within finite time with probability 1 under certain assumptions on the behavior of the system" [@bracha1983resilient]. This can be achieved by adding characteristics of randomness to the protocol, such as random retry timeouts for failed messages, or by the application of _unreliable failure detectors_ as in the Chandra–Toueg consensus algorithm [@chandra1996unreliable]. Additionally, adding pseudo-synchronous behavior like in Raft (which is described in detail in the [following section](#sec:raft)), where all messaging goes in rounds by enforcing an enumeration by terms and indexes, removes the initial problem of asynchronous consensus.

\todo{Relation to Atomic Broadcasts (Equivalency to Byzanzine Fault Tolerant Consensus)?}

<!--
in systems with crash failures, atomic broadcast and consensus are equivalent problems. 
A value can be proposed by a process for consensus by atomically broadcasting it, and a process can decide a value by selecting the value of the first message which it atomically receives. Thus, consensus can be reduced to atomic broadcast. [@chandra1996unreliable]
Conversely, a group of participants can atomically broadcast messages by achieving consensus regarding the first message to be received, followed by achieving consensus on the next message, and so forth until all the messages have been received. Thus, atomic broadcast reduces to consensus. This was demonstrated more formally and in greater detail by Xavier Défago, et al.[https://doi.org/10.1145%2F1041680.1041682]

The Chandra-Toueg algorithm[6] is a consensus-based solution to atomic broadcast.

The Zookeeper Atomic Broadcast (ZAB) protocol is the basic building block for Apache ZooKeeper, a fault-tolerant distributed coordination service which underpins Hadoop and many other important distributed systems.[8][9]

-->

TODO difference between decentralized (raft, paxos... (but care that single leader may be a problem)) and centralized consensus (zookeeper?)

Distributed consensus protocols describe a procedure to reach a common agreement among nodes in a distributed or decentralized multi-agent system.

"we apply consensus to ensure reliability and fault tolerance..."

Popular examples for distributed consennsus protocols are Paxos (subsection [@sec:paxos]) and Raft (section [@sec:raft]), the latter being the focus of this work.

Example applications where consensus is needed:
- Clock synchronisation
- Google PageRank
- Smart Power Grid
- Load Balancing
- Key-Value Stores
- and ultimatively Blockchain
- \todo{Find other consensus examples then the wiki ones}
- Distributed systems in general

<!-- (https://en.wikipedia.org/wiki/Consensus_(computer_science)) -->

Different ways to categorize consensus protocols. One is by the size of the value scope:

**Single-Value Consensus Protocols**
- such as Paxos
- nodes agree on a single value

**Multi-Value Consensus Protocols**
- such as Multi-Paxos or Raft
- agree on not just a single value but a series of values over time
- forming a progressively-growing history
- may be achieved naively by running multiple iterations of a single-valued consensus protocol
- but many optimizations and other considerations such as reconfiguration support can make multi-valued consensus protocols more efficient in practice

The other way to categorize those protocols is by fault tolerance (see [Types of Possible Faults](#sec:possible-faults) for reference):

**Crash-Fault Tolerant (BFT) Consensus Protocols**

For Fail-Stop failure model...

"A consensus protocol tolerating those halting failures must satisfy the following properties:

\todo{Re-Research and rephrase}

<!--
 Termination: All non-faulty processes must eventually decide on a value
• Agreement: All non-faulty processes agreee on same value
• Validity: Agreed upon value must be the same as the initial proposed “source” value -->

**Termination**
Eventually, every correct process decides some value.
**Integrity**
If all the correct processes proposed the same value $y$, then any correct process must decide $y$.
**Agreement**
Every correct process must agree on the same value."

For a faulty system to still be able to reach consensus, even more nodes are required than initially for a system to be [fault-tolerant](#sec:possible-faults):

$\left \lceil (n + 1)/2 \right \rceil$ correct processes are necessary and sufficient to reach agreement [@bracha1983resilient], or in other words $n \geq 2k + 1$ where $k$ is the number of crashed processes and $n$ the number of total processes/nodes needed to tolerate this number of faulty nodes.

"there is no consensus protocol for the fail-stop case that always terminates within a bounded number of steps" [@bracha1983resilient] as shown above... need randomness 

**Byzantine-Fault Tolerant (BFT) Consensus Protocols**


\todo{Illustration for byzantine generals problem like in https://www.sciencedirect.com/topics/computer-science/byzantine-fault}

The _Byzantine Generals Problem_ is a classic problem in distributed systems that is not as easy to implement, adapt, and understand as it might seem to a systems architect [@lamport1982byzantine]

BFT consensus is defined by the following four requirements:

\todo{Rephrase}

**Termination**: Every non-faulty process decides an output.
**Agreement**: Every non-faulty process eventually decides the same output $y$.
**Validity**: If every process begins with the same input $x$, then $y = x$.
**Integrity**: Every non-faulty process' decision and the consensus value $y$ must have been proposed by some nonfaulty process.

For any consensus protocol to attain these BFT requirements, $\left \lceil (2n + 1)/3 \right \rceil$ correct processes/nodes are necessary and sufficient to reach agreement, or in other words $n \geq 3k + 1$ where $f$ is the number of Byzantine processes and $n$ the number of total processes/nodes needed to tolerate this number of faulty nodes. This fundamental result was first proved by Pease, Lamport et al. [@pease1980faults] and later adapted to the BFT consensus framework [@bracha1983resilient].

\todo{Chart on byzantine quorums from https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L9-bft.pdf}

\todo{Mention this is not the focus of this work}

Byzantine-Fault tolerant (BFT) consensus protocols are naturally Crash-Fault tolerant (CFT) [@xiao2020survey]


"TODO Use cases: Recent advances in permissioned blockchains [58, 111, 124], firewalls [20, 56], and SCADA systems [9, 92] have shown that Byzantine fault-tolerant (BFT) state-machine replication [106] is not a concept of solely academic interest but a problem of high relevance in practice. [@distler2021byzantine]"

\todo{describe that practical consensus protocols need to incorporate a reconfiguration protocol (and describe what this means)}

TODO describe the situation of weighted quorums, for example as it is in Proof-of-Stake

TODO describe reconfiguration after network partitions

TODO typically communicate with a heartbeat (see HA clusters)

#### State Machine Replication

State machine replication is... [@schneider1990statemachine] and can be achieved through consensus protocols. 

Strong consistent!

\todo{Are all state-machine approaches non-byzantine? I don't think so. What about byzantine raft/paxos?}

Single leader:
Single master computing means somehow we order the changes.
The order can come from a centralized master or some Paxos-like
[11] distributed protocol providing serial ordering

state machine == derived aggregate. Log == trivial facts [@sec:calm]
leveraging monotonicity and append-only characteristics as shown in [@sec:consistency-decisions]

\todo{Rephrase}

Consensus in distributed computing is a more sophisticated
realization of the aforementioned distributed system. In a
typical distributed computing system, one or more clients
issue operation requests to the server consortium, which
provides timely and correct computing service in response
to the requests despite some of servers may fail. Here the
correctness requirement is two-fold: correct execution results
for all requests and correct ordering of them. According to
Alpern and Schneider’s work on liveness definition [31] in
1985, the correctness of consensus can be formulated into
two requirements: **safety** — every server correctly executes the
same sequence of requests, and **liveness** — all requests should
be served.
To fulfill these requirements even in the presence of faulty
servers, server replication schemes especially state machine
replication (SMR) are often heralded as the de facto solution.
SMR, originated from Lamport’s early works on clock synchronization in distributed systems [@lamport1978time], [33], was formally
presented by Schneider [@schneider1990statemachine] in 1990. Setting in the clientserver framework, SMR sets the following requirements:
1) All servers start with the same initial state;
2) Total-order broadcast/atomic broadcast: All servers receive the same sequence of requests as how they were generated from
clients;
3) All servers receiving the same request shall output the
same execution result and end up in the same state. 
[@xiao2020survey]

TODO cite the slides for the State-Machine Approach for fault-tolerance https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/state-machine.pdf}
  \caption{Illustration of the messaging between clients and cluster in state-machine replication}
  \label{fig:state-machine}
\end{figure}

TODO there are both BFT and CFT state machine replication protocols: 
"Recent advances in permissioned blockchains [58, 111, 124], firewalls [20, 56], and SCADA systems [9, 92] have shown that Byzantine fault-tolerant (BFT) state-machine replication [106] is not a concept of solely academic interest but a problem of high relevance in practice. [@distler2021byzantine]"

<!-- https://en.wikipedia.org/wiki/State_machine_replication -->

<!--
#### Other Kinds of Protocols

- Master-Slave
- Active-Passive (any difference to master-slave?^)
- Primary-Copy
- TODO Mention them here or just below?
-->

#### Read One / Write All (ROWA)

The simplest, yet weakest approach.
"Update anywhere, anytime, anyhow" - Lazy Group ? 
- Simplest approach (simpler than primary copy)
- No availability if one node crashes (cannot write)
  - strong dependency of a write on the availability of all replicas
  - As client writes itself, transaction time increases linearly by number of nodes
- Reading is easy and cheap

#### Primary-Copy Replication

"Update anywhere-anytime-anyway transactional replication has unstable behavior as the workload scales up: a ten-fold increase in nodes and traflc gives a thousand fold increase in deadlocks or reconciliations. Master copy replication (primary copyj schemes reduce this problem." https://dl.acm.org/doi/pdf/10.1145/233269.233330

Another simple, but weak approach. One primary server, N backup (copy) servers.

See section "11.6 Replicated state machines vs. primary copy approach" from https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf

TODO incorporate the table State-machine vs. Primary-backup at the end of the slides https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

TODO different use cases then consensus: Not primarily for fault-tolerance, but to provide disaster recovery (having backups) or local copies of the data (to improve performance + availability, e.g. for mobile users)

Primary copy server can easily become a bottleneck in the system. It does not scale with the number of nodes and has a detrimental impact especially if faulty behavior appears. What happens if the primary copy server fails? Failure detection mechanisms are required to detect such situations and subsequently select an alternate primary copy server, creating the risk of multiple primary copies when different network partitions occur; a problem that is mitigated in consensus protocols, as described in the corresponding sections.

#### Optimistic Replication {#sec:optimistic-replication}

There are many use cases where the cost of replication is too high when synchronization between nodes in a replica set is required, especially for real-time live collaboration applications or applications that cannot be guaranteed to be online all the time (such as mobile apps). In these cases, the consistency requirements should be attenuated to eventual consistency so that the application can be designed in a non-blocking manner. One approach to solving problems of this scope is _optimistic replication_, as described by Shapiro et al. [@shapiro2005optimistic]. In optimistic replication, any client managing a replicated state is allowed to read or update the local replica at any time.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.9\textwidth]{images/crdts-optimistic-replication.pdf}
  \caption{Schematic communication model between the nodes in optimistic replication for a single operation. A write is acknowledged immediately and non-blocking, without synchronization with other nodes. It is then eventually propagated to other nodes. In the event of a write conflict, the conflict is automatically resolved in the background using either a decentralized protocol or a centralized entity. Once the conflict is resolved, the actual agreed value is written back to the node's internal state and acknowledged.}
  \label{fig:crdts-optimistic-replication}
\end{figure}

Such local updates are only tentative and their final outcome could change, as they may conflict with an update from another client. In optimistic replication, conflicts should not be visible to the user (since conflict resolution requiring human intervention would compromise consistency, which is undesirable in the intended use cases), so they must be resolved in the background, either by a decentralized protocol or a central server instance. The replicas are allowed to differ in the meantime, but they are expected to eventually become consistent. Figure \ref{fig:crdts-optimistic-replication} illustrates this behavior. Note that it only illustrates it for a single operation. In case of multiple operations, reordering the operations is also part of the conflict resolution, depending on the protocol.

Optimistic replication is especially suitable for problems that meet the monotonicity requirements as described in the CALM theorem (cf. subsection [@sec:calm]), as this class of problems requires no conflict resolution at all.

\todo{Describe steps of OR: server/decentralized protocol defines order of events...}

\todo{Possible to use CRDTs for chronicledb TAB+ tree?}

An example of optimistic replication for strong eventual consistency are _conflict-free replicated data types_ (CRDT). CRDTs were introduced by Shapiro et al. and describe data types with a data schema and operations on it that will be executed in replicated environments to ensure that objects always converge to a final state that is consistent across replicas [@shapiro2009crdts]. CRDTs ensure this by requiring that all operations must be designed to be conflict-free, since CRDTs are intended for use in decentralized architectures where operation reordering is difficult to achieve. Therefore, any operation must be both _idempotent_ (the same operation applied multiple times will result in the same outcome) and _commutative_ (the order of operations does not influence the result of the chained operations), which also requires them to be free of side effects [^secro].

CRDTs take the consistency problem onto the level of data structures. They make use of optimistic replication to allow for such operations without the need of replicas to coordinate with each other (often allowing updates to be executed even offline) by relying on merge strategies to resolve consistency issues and conflicts, once the replicas synchronize again. They are designed in a way that the overall state resolves automatically, becoming eventually consistent. Due to the nature of the operations to be idempotent, CRDTs can only be used for a specific set of applications, such as distributed in-memory key-value stores like Redis, which is commonly used for caching [@redis2022crdts]. Another common set of use cases are realtime collaborative systems, such as live document editing (relying on suitable data structures to hold the characters with good time complexity properties for both random inserts and reads, such as an _AVL tree_ [@adelsonvelskii1963algorithm]) [@shapiro2009commutative]. CRDTs are not suitable for append-only logs, streams or event stores, as designing an operation to append a record or event with idempotency and commutativeness is too expensive.

[^secro]: Recent research is trying to find such a replicated data type that does not require operations to be commutative. One approach describes _Strong Eventually Consistent Replicated Objects_ (SECROs), aiming to build such a data type with the same dependability properties as CRDTs [@de2019generic]. The authors achieve this by ensuring a total order of operations across all replicas, but still without synchronisation between the replicas: they show that it is possible to order the operations asynchronously.

CRDTs are not the only way to achieve optimistic replication. Before CRDTs, _operational transformations_ (OT) were the default approach for solving these kinds of problems, at least in academia. Due to their complexity and the difficulty to implement them and also to write formal proofs for different sets of operations needed in practical applications [@li2010admissibility], they were often replaced by alternative approaches like CRDTs, so we will not discuss them in detail in this work.

\todo{Grammar checking}
In theory, CRDTs are designed for decentralized systems in which there is no central authority to decide the end state. In practice, there is oftentimes a central instance, such as in SaaS offerings on the web. Decentralized conflict resolution is no longer a requirement for those systems, so CRDTs could be too heavy-weight. Moving the conflict resolution to a central instance (which actually could be a cluster of nodes with stronger consistency guarentees) reduces complexity in the implementation of optimistic replication. This is actually the case in multiplayer video games—especially massive multiplayer online games (MMOGs). They introduce a class of problems that need techniques to synchronize at least a partial state between a lot of clients. Strong consistency would cause this games to experience bad performance drops, since a game's client wouldn't be able to continue until it's state is consistent across all subscribers to that state, so the way to go is eventual consistency. One can learn a lot from these approaches and adopt it to other real-time applications, such as Figma did for their collaborative design tool, comprehensively described in a blog post [@figma2019multiplayer].

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
  \caption{In the coordination-free transaction and replication protocol Eris, communication happens in a single round-trip in the normal case, while the transactions are consistently ordered by a sequencer on the network layer}
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

#### Concrete Protocols

This subsection provides an overview of concrete, popular replication protocols to the reader. Some of them are used in production for years, while others are subject of academic studies only.

The [following subchapter](#sec:raft) will then spend a closer look on Raft, a consensus-based state machine replication protocol. The author has chosen to use Raft for replication of the event store which is the subject of this thesis.

##### Paxos {#sec:paxos}

"Paxos is one of the most widely deployed consensus algorithms today (blockchains excluded).
Several Google systems use Paxos, including the Chubby [@burrows2006chubby] lock service and the Megastore [5] and Spanner [20] storage systems"

- State machine replication: Executes the same set of commands in the same order, on multiple participants
- Strong consistent
- Single-Value, but also multi-value using Multi-Paxos (which is therefore the most commonly used version of Paxos)
- Had been state of the art for a long time for strong consistent replication
- Based on distributed consensus
- original by Leslie Lamport [@lamport1998paxos] and further improvements by himself [@lamport2006fast]
- although he tried to make it understandable [@lamport2001paxos], it is shown that people of any kind (students, engineers...) still struggle to understand it in its fullest [@ongaro2013raft] 
- Formally verified using TLA+, as attached to the paper [@lamport2006fast], and also by thirds for Multi-Paxos [@chand2016formal]
- TLA+ was also invented by Lamport
- handles fail-stop but not Byzantine failures
- does it has a reconfiguraion protocol?

##### ZooKeeper Atomic Broadcast (ZAB)

TODO describe zookeeper here in short (mostly just point to references), and cite it in the paragraph on Kafka

##### Viewstamped Replication

- Primary-Copy https://pmg.csail.mit.edu/vr/oki88vr-abstract.html
- State machine replication?????
- offers a reconfiguration protocol
- handles fail-stop but not Byzantine failures

ABC [@oki1988viewstamped], [@liskov2012viewstamped]

##### Practical Byzantine Fault Tolerance (PBFT)

[@castro1999practical]

<!-- 700BFT,  Query/Update (Q/U) and Hybrid Quorum (HQ) -->

##### Leaderless Byzantine State Machine Replication {#sec:leaderless}
SMR in general means that there is a single leader (as in Paxos and Raft...)...


Geo-replication... Due to their reliance on a leader replica, classical SMR protocols offer
limited scalability and availability under a geo-replicated setting. To solve this problem, recent protocols follow instead a leaderless approach, in which each replica is able
to make progress using a quorum of its peers. These new leaderless protocols
are complex and each one presents an ad-hoc approach to leaderlessness.

there are multi-leader approaches (TODO reference multi-leader raft) and most recent literature deals with leaderless SMR like Tempo [@enes2021tempo] or Wintermute [@rezende2021leaderless]...

Leaderless SMR shows its strengths especially when used in blockchains... high cost of replication... especially when strong consistent... see [Blockchain Consensus Protocols](#sec:blockchain-consensus)

##### Blockchain Consensus Protocols {#sec:blockchain-consensus}

In recent years, the problem of byzantine fault tolerant consensus has raised significantly more attention due to the widespread success of blockchains and blockchain-based applications, especially cryptocurrencies such as Bitcoin, which successfully solved the problem in a public setting without a central authority...

Blockchains are distributed systems par excellence... Using strong consistent consensus protocols to ensure every node reads the same... Handling not only fail-stop, but especially byzantine faults is crucial to those consensus protocols to secure a blockchain.

As shown in subsections [@calm] and [@consistency-decisions], monotonicly growing append-only systems can be replicated with eventual consistency in a coordination-free way. Blockchains are distributed ledgers that provide this monotonic characteristics... (do they all?) So, depending on the requirements of the chain, a great range of consistency levels are possible: strong consistency with coordination between nodes to agree not only on a final state, but also to output the same order of blocks and to only expose the ledger state when all are ready to expose... or eventual consistency are possible (TODO show examples for both)

"Within a Blockchain network system, the
strong consistency model means that all nodes have the same ledger at the same time, and during the time when the
distributed ledger is being updated with new data, any subsequent read/write requests will have to wait until the
commit of this update"

" The consistency property has raised some controversial debate. Some argue
that Bitcoin systems only provide eventual consistency [6], which is a weak consistency. Others claim that Bitcoin guarantees strong consistency, not eventual consistency [7]." [@kanga2020management]

"Within a Blockchain network system, the
strong consistency model means that all nodes have the same ledger at the same time, and during the time when the
distributed ledger is being updated with new data, any subsequent read/write requests will have to wait until the
commit of this update"

BTC seeems to be sequentially consistent, as all nodes must agree on the same sequence, but only in program order, and not all must see transactions at the same time, and the consistency is controlled with the mining difficulty in the PoW consensus...

"certain Blockchain applications are less risk-averse and may benefit from a weaker consistency guarantee for convenience and performance. "

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


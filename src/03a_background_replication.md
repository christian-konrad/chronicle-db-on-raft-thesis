## Replication {#sec:why-replication}

\epigraph{A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable.}{--- \textup{Leslie Lamport}}
<!-- [@milojicic2002discussion] -->

<!-- ### why replication is neccessary, and what it is-->

This section provides the reader with an overview of the topic of distributed systems, defines the term _replication_, explains why replication is crucial for many of those systems, discusses various replication protocols, and concludes with a literature review of recent research in this area. The goal of this section is to provide the reader with a thorough understanding of what replication is, why, where, and when it is needed, and how replication protocols work.

\todo{Glossary?}

A _distributed system_ is a collection of autonomous computing elements that appears to its users as a single coherent system [@steen2007distributed]. To achieve this behavior, such a system needs to offer certain levels of _availability_, _consistency_, _scalability_ and _fault-tolerance_; therefore, many modern distributed systems rely heavily on replicated data stores that maintain multiple replicas of shared data [@liu2013replication]. Depending on the replication protocol, the data is accessible to clients at any of the replicas, and these replicas communicate changes to each other using message passing.

Thus, a distributed database is a distributed system that provides read and write access to data. Replication in these databases takes place to varying degrees: from simple replication of metadata and current system state to full replication of all payload data, depending on the requirements and characteristics of the system.

Many modern applications, especially those served from the cloud, are inherently distributed. With edge computing, such applications can be distributed not only to nodes in one or more clusters serving many clients, but also to all of those clients participating in the edge-cloud network. Users of cloud applications expect the same experience independent of their geographic region and also regardless of the amount of data they produce and consume.

In complex distributed systems, there are multiple interoperating replicated _(micro)services_ with different availability and consistency characteristics. Ensuring that these systems and services can reliably work together is the topic of container orchestration (e.g., through Kubernetes [@kubernetes2022github]) and service composition [@nikoo2020survey; @camunda2022zeebe]—which need to be replicated, too—which we will not discuss in this work.

Example use cases for replication in distributed systems (additionally to those mentioned in the [Introduction](#sec:introduction)) include large-scale software-as-a-service applications with data replicated across data centers in geographically distinct locations [@mkandla2021evaluation], applications for mobile clients that keep replicas near the user's location to support fast and efficient access [@silva2022geo], key-value stores that act as bookkeepers for shared cluster metadata [@etcd2022], highly available domain name systems (DNS) [@bi2020craft], blockchains and all their various use cases (see [Blockchain Consensus Protocols](#sec:blockchain-consensus)), or social networks where user content is to be distributed cost-effectively to millions of other users [@khalajzadeh2017cost], to name a few.

The following subsections describe the fundamental concepts of _dependable_ distributed systems, as well as use cases for replication and the challenges in these particular cases, before outlining relevant replication protocols.

### Horizontal Scalability

\todo{Verify phrasing of this subsection}

Standalone applications can benefit from _scaling up_ the hardware and leveraging concurrency techniques/multithreading behavior (in this case we speak of _vertical scalability_), but only to a certain physical hardware limit (such as CPU cores, network cards or shared memory). Approaching this limit, these applications and services can no longer be scaled economically or at all, as the hardware costs increase dramatically. In addition, applications running on only one single node pose the risk of a single point of failure, which we want to counter with replication.

To scale beyond this limit, a system must be designed to be _horizontally scalable_, that is, to be distributable across multiple computing nodes/servers (also known as to _scale out_). Ideally, the amount of nodes scales with the amount of users, but to achieve this, certain decisions must be made regarding [consistency models](#sec:consistency) and [partitioning](#sec:partitioning). Legacy vertically scalable applications cannot be made horizontally scalable without rethinking the system design, which includes replication and message passing protocols between the nodes to which the application data is distributed.

With vertical scaling, when you host a data store on a single server and it becomes too large to be handled efficiently, you identify the hardware bottlenecks and upgrade. With horizontal scaling instead, a new node is added and the data is partitioned and split between the old and new nodes.

\todo{Illustration of vert. vs horz. scalaing}
TODO this chart https://www.cloudzero.com/hubfs/blog/horizontal-vs-vertical-scaling.webp

With an increasing number of users served, the number of nodes in a horizontally scalable system grows. In the ideal case, the number of nodes required to keep the perceived performance for users constant increases linearly with the number of distinct data entities served (such as tables or data streams with distinct schemas) [@williams2004web]; in other words, the transaction rate grows linearly with the computational power of the cluster. This is not neccessarily the case when a increasing number of users access the same stream/table or inter-node/inter-partition transactions are executed. In the latter case, vertical scaling or advanced partitioning techniques [@aaqib2019efficient] still come handy. In practise, many applications achieve near-linear scalability.

\todo{NTH: show chart and math for near-linear scalability if time}

\todo{Chart: vertical vs horicontal scalability (i.e. the zeebe one)}

Chart from https://www.liquidweb.com/blog/horizontal-vs-vertical-scaling/

Opting in for horizontal scaling is beneficial for large-scale cloud and edge applications as it is easier to add new nodes to a system instead of scaling up existing ones, even if the initial costs are higher as the infrastructure and algorithms must be implemented in first place.

The system design that enables horizontal scalability is also a key requirement for replication. To achieve fault tolerance and availability through replication, it is required to store replicas of a dataset on multiple nodes, therefore the book keeping and message passing infrastructure that is neccessary for partitioning is also a basic requirement for replication protocols [@liu2013replication].

### Safety and Reliability

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

A high degree of reliability, while necessary, is not sufficient to ensure safety. In fact, there can even be a tradeoff between safety and reliability. To understand this, we need a different way to model reliability and safety. We look at the probabilities of three possible types of responses to a client request to the system:

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

Another type of failure that is important to address are _disasters_ in the data center. _Disaster recovery_ includes all actions taken when a primary system fails in such a way that it cannot be restored for some time, including the recovery of data and services at a secondary, surviving site. One important disaster recovery measure is _geo-replication_, which can be used to manage disasters of the type that render data centers unavailable due to, for example, a natural disaster such as a flood or earthquake. Geo-replication is discussed in more detail [later in this paper](#sec:geo-replication). Since disaster recovery is a complex subject area of its own that goes beyond the application of replication mechanisms, it is not discussed in detail in this thesis (apart from geo-replication).

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

An availability of 99.999 % is referred to as _five nines_ and is the typical standard for telecommunication networks and a hit target for many site reliability engineering (SRE) departments of service providers, while popular commercial platforms provide lower levels of availability, such as Amazon Web Services (AWS), which provides 99.99% availability for their standard offering of S3 [@amazon2020s3availability]. Oftentimes, such services are geo-replicated and deployed in specific _availability zones_ so that they are available during peak operating hours for the corresponding time zones. The maintenance time (e.g. for updates and general MTTR) is then usually scheduled for the night hours. For critical business operations, a certain level of availability must be guaranteed by the service providers, which they typically specify in their _Service Level Agreements_ (SLAs). If the service provider fails to keep these availability promises, the provider declares themselves responsible to compensate for the damage incurred (the missed revenue). The confidence of having an excellent SLA that ensures high availability, among other things, can often be a decisive competitive advantage. For example, the SLA for AWS S3 guarantees customers tiered compensation based on the availability achieved in the corresponding billing cycle. As of the time of writing, the total cost of the billing cycle will be refunded if the service didn't met an availability of at least 95 % throughout this cycle [@amazon2022s3sla].

When orchestrating multiple services in the context of a _service oriented architecture_ (SOA), the availability of a resulting aggregated system or business process depends heavily on all the orchestrated services as well as the orchestrating system itself, and decreases for each participating system. Ignoring network availability, overlapping unavailability periods and other factors, and assuming all services are run in a sequential order, we can roughly estimate the overall availability as

$$ A_{\textrm{serial}}(t) \approx \prod_i A(t)_i $$

where $A(t)_i$ are the availabilities of the various systems in the period of $t$, including the orchestration engine. Note that this is just a naive estimation that does not take into account failover and compensation mechanisms of the orchestration engine that can actually improve the availability again. Figure {fig:serial-availability} illustrates this for two nodes.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/sequential-service-availability.pdf}
  \caption{Example for the resulting availability of a sequential orchestration of services}
  \label{fig:serial-availability}
\end{figure}


Conversely, replication is a way to drastically increase the availability of a system. By having more than one copy of important information, the service continues to be usable even when some copies are inaccessible. By providing a _High-Availability Cluster_ (HA Cluster)[^load-balancing] of replicas, the probability of failure of the whole cluster during a fixed period of time decreases with every replica, decreasing the estimated annual downtime. When we ignore factors like failures of the replication system itself, faults in the underlying network, and the time needed for cluster reconfigurations on node failures (as included in the cluster MTTR), we can model the availability of the cluster naively as

$$ A_{\textrm{cluster}}(t) \approx 1 - (1 - A(t))^n = 1 - U(t)^n $$

where $n$ is the number of replicas, $A(t)$ is the availability of a single node in the period of $t$, and $U(t)$ the respective unavailability. A replica configuration like this fails if all of its replicas fail. Figure {fig:parallel-availability} illustrates this for two nodes.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/replicated-node-availability.pdf}
  \caption{Example for the resulting availability of a replica set}
  \label{fig:parallel-availability}
\end{figure}

[^load-balancing]: Load balancing is another technique for achieving high availability through service replication. Since this work only deals with data replication, this topic is not covered here (although some session data must be replicated between these services if they are not stateless in the first place).

As shown, the common way to achieve high availability is through the replication of data in multiple service replicas. High availability comes hand-in-hand with fault-tolerance, as services remain operational in case of failures as clients can be relayed to other working replicas. 

### Consistency {#sec:consistency}

Building scalable and reliable distributed systems requires a trade-off between consistency and availability. Consistency is a property of the distributed system that ensures that every node or replica has the same view of the data at a given point in time, regardless of which client updated the data. Ideally, a replicated data store should behave no differently than one running on a single machine, but this often comes at the cost of availability and performance. Deciding to trade some consistency for availability can often lead to dramatic improvements in scalability [@pritchett2008base].

Consistency is an ambiguous term in data systems: in the sense of the _ACID model_ (Atomic, Consistent, Isolated and Durable), it is a very different property than the one described in the _CAP theorem_ (we explain this later in this section). The ACID model describes consistency only in the context of database transactions, whereas consistency in the CAP model refers to a single request/response sequence of operations. We focus on the definition in the CAP theorem in this work. In the distributed systems literature in general, consistency is understood as a spectrum of models with different guarantees and correctness properties, as well as various constraints on performance.

A database consistency model determines the manner and timing in which a successful write or update is reflected in a subsequent read operation of that same value. It describes what values are allowed to be returned by operations accessing the storage, depending on other operations executed previously or concurrently, and the return values of those operations. There exists no one-size-fits-all approach: it is difficult to find one model that satisfies all main challenges associated with data consistency [@mkandla2021evaluation]. 

There are two distinct perspectives on consistency models: the data-centric and client-centric perspective, as illustrated in figure \ref{fig:data-client-centric-perspective} [@bermbach2013towards]. The data-centric perspective analyzes consistency from a replica's point of view, where the distributed system synchronizes read and write operations of all processes to ensure correct results. Similarly, the client-centric perspective examines consistency from a client's point of view. From this perspective, it is sufficient for the system to synchronize only the data access operations of the same process, independently of the others, to ensure their consistency. This is justified because common updates are often rare and mostly access private data [@campelo2020brief]. It is sufficient to _appear_ consistent to clients, while being at least partially inconsistent between cluster nodes. Therefore, client-centric consistency models are in general weaker than data-centric models. 
<!-- In this work, we focus on the data-centric perspective because the client-centric perspective depends on it and can be derived from it. -->

\begin{figure}[h]
  \centering
  \includegraphics[width=1\textwidth]{images/data-client-centric-consistency.pdf}
  \caption{The data-centric and client-centric perspective on consistency models, as described by Bermbach et al.}
  \label{fig:data-client-centric-perspective}
\end{figure}

\todo{Hierarchical overview of the consistency levels, with ascending weakness}

Following, a few of those consistency models are discussed in the order of their relevance to this work [@steen2007distributed]. Throughout the discussion, we use this notation to illustrate the behavior of the models by examples:

$w(x, v)$ denotes a successful write operation of the value $v$ into the variable $x$.

$r(x) = v$ denotes a read of $x$ that returns the value $v$.

\todo{Work finished to this point.}

\todo{Rephrase}

<!--

Models get weaker from top to bottom

##### Data-centric

Strong consistency: After a successful write (incl. ACK), the value can be read at all nodes immediately

Sequential: Lamport 1979; no immediate read, but writes have to be seen in the same order at every node

Causal: Only causally related writes must be seen in the same order at every node

Weak: Only writes that are explicitly synchronized are consistent. Everything else in between is unordered (but at least the same set of operations)

##### Client-centric

Eventual: if no update takes a very long time, all replicas eventually become consistent. Eventually consistency means in the absence of updates all replicas converge toward identical copies.

##### Need explicit, manual synchronization mechanisms

TODO search for a matrix or table for all consistency models 

-->

\todo{Denote everything mathematically if sufficient time and if it does not unneccessary complexity}

<!-- Strong Consistency -->

\paragraph{Strong Consistency.} Strong consistency is, from the view of a client, the ideal consistency model: a read request made to any of the nodes of the replica cluster should return the same data. A replicated data store with strong consistency therefore behaves indistinguishably from one running on a single machine. A strong consistency model provides the highest degree of consistency by guaranteeing that all operations on replicated data behave as if they were performed atomically on a single copy of the data. After a successful write operation, the written object is immediately accessible from all nodes at the same time: _what you write is what you will read_[^read-after-write]. However, this guarantee comes at the cost of lower performance, reliability, and availability compared to weaker consistency models [@attiya1994sequential; @liu2013replication]: Algorithms that guarantee strong consistency properties across replicas are more prone to message delays and render the system vulnerable to network partitions, if not handled properly. Such algorithms do need special strategies to provide partition-tolerance to the cost of availability.

[^read-after-write]: For this reason, it is sometimes referred to as _read-after-write consistency_.

Strong consistency is mandatory for use cases with zero tolerance for inconsistent states, such as banking. To achieve strong consistency, an absolute global time order must be maintained [@lamport1978time]. A good example is linearizability 

TODO explain how to achieve strong consistency: Examples like linea..., 2PC, acks...

\todo{Timeline diagram}

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/strong-consistency-flow.pdf}
  \caption{Communication model between clients and replica cluster nodes to ensure sequential consistency. A write is acknowledged only when it is consistent throughout all cluster nodes.}
  \label{fig:strong-consistency-flow}
\end{figure}

TODO what about strict and sequential? linearizability ? It seems like all of those are strong consistency models, so they are subclasses. Strict is not used in practice as it is inpractical, so in practice, strong = sequential.

Sequential: 
"However, note that it does not always necessarily hold true that the specified global ordering is the same as the real-time sequence of execution of each operation."
Mostly used in practice to achieve a strong level of consistency
Does not care about realtime properties, as strict consistency, which expects a global ordering by time. This means that all reads at all nodes expose the same order of writes for sequential consistency, but not neccessarily in the absolute order (by timestamps) that clients requested the reads, which is impractical and needs reordering at and between nodes if concurrent writes appear in the wrong order
(sequential is sufficient for our implementation; OOO handling of the event store itself should be more efficient than existing strict consistency mechanisms)
(Raft is sequential)!

"absolute perfect consistency is termed as Strict Consistency but is impractical to implement and hence is reduced to a theoretical basis only. Both of the levels are discussed in detail below."

Strong consistent data stores are easy to use for developers, as they do not need to be aware of differences in replicas of the given data store. The whole system appears as it would be one single node to the user. 

- TODO more [@gotsman2016cause]
"requesting stronger consistency in too many places may hurt performance, and requesting it in too few places may violate correctness"

TODO data-centric

"Ideally, we would like replicated databases to provide strong consistency, i.e., to behave as if a single centralised node handles all operations. However, achieving this ideal usually requires synchronisation among replicas, which slows down the database and
even makes it unavailable if network connections between replicas fail (= network partitions)". This does not apply to all protocols with strong consistency: Raft, the protocol of choice of this work, allows the system to be responsive even when the network is partitioned under certain constraints [TODO raft paper].

"For this reason, modern replicated databases often eschew synchronisation completely; such databases are commonly dubbed eventually consistent [47]. In these databases, a replica performs an operation requested by a client locally without any synchronisation with other replicas and immediately returns to the client; the effect of the operation is propagated to the other replicas only eventually. This may lead to anomalies—behaviours deviating from strong consistency"

**Eventual Consistency** - [@vogels2009eventually]
Client-centric

Eventual consistency makes sure that data of each node of the database gets consistent eventually. Time taken by the nodes of the database to get consistent may or may not be defined.

Eventually consistency means in the absence of updates all replicas converge toward identical copies.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/eventual-consistency-flow.pdf}
  \caption{Schematic communication model between clients and replica cluster nodes for eventual consistency. A write is acknowledged immediately before it is propagated consistently across all cluster nodes.}
  \label{fig:eventual-consistency-flow}
\end{figure}

"Because of the possibility of inconsistency, the application developer must be explicitly in the knowledge of the replicated nature of data items in the data store. Compared to strong consistency, the developer must consider ..."

"strong consistency is the most expensive one to achieve. This is mainly due to the requirements for a 2 phase commit (2PC) type of distributed transaction. 2PC transactions have poor performance, limited scalability and require tight coupling of services. modern microservice architectures oftentimes chose eventual consistency for the consistency outcome"
- TODO list applications
- Client-centric
"This model states that all updates will propagate through the system and all replicas will gradually become consistent, after all updates have stopped for some time [56, 60]. Although this model does not provide concrete consistency guarantees, it is advocated as a solution for many practical situations [10–12, 24, 60] and has been implemented by several distributed storage systems [21, 28, 35, 43]."
- Can increase performance dramatically; see for example kubernetes made eventual consistent instead of strong to meet edge-computing requirements [@jeffery2021rearchitecting]
- Not always neccessary; good system design allows even strong consistent services to work at the edge 

"Whereas eventual consistency is only a liveness guarantee (updates will be observed eventually), strong eventual consistency (SEC) adds the safety guarantee that any two nodes that have received the same (unordered) set of updates will be in the same state."

\todo{BASE? [@pritchett2008base]}

Basically Available, Soft state, Eventually consistent

TODO NoSQL Databases are often BASE and not ACID and therefore not strong consistent...

  - AWS services like S3 where eventual consistent when introduced
  - Maximizing read throughput
  - Returning recent writes is not guaranteed
  - Eventual different responses from different nodes at different points in time

Eventual consistency be a problem when users don't receive on a read what they've just written, and they repeat the operation and it is not idempotent. For this, there is the model of strong eventual consistency... which enforces idempotency and commutativeness on all operations... An example of a use case for strong eventual consistency are _conflict-free replicated data types_ (CRDT), which are described in (TODO section here)


**Causal Consistency** - [@shen2015causal]
- Data-centric
"Causal consistency is the strongest form of consistency that satisfies low Latency, defined as the latency less than the maximum wide-area
delay between replicas [@lloyd2011don]."

"Causal Consistency means that all operations that are causally related must be serialized in the same order on all replicas. An operation O is causally dependent on an operation P if one or more of the following conditions hold:

O and P were both triggered by the same process and P was chronologically prior to O.
O is a read, P is a write, and O read the result of P.
O is causally dependent on an operation X, which in turn is causally dependent on P (transitivity)."

"Causal consistency is a model in which a sequential ordering is maintained only between requests that have a causal dependency. Two requests A and B have a causal dependency if at least one of the following two conditions is achieved: (1) both A and B are executed on a single thread and the execution of one precedes the other in time; (2) B reads a value that has been written by A. Moreover, this dependency is transitive, in the sense that, if A and B have a causal dependency, and B and C have a causal dependency, then A and C also have a causal dependency [56, 60]. Thus, in a scenario of an always-available storage system in which requests have causal dependencies, a consistency level stricter than that provided by the causal model cannot be achieved due to trade-offs of the CAP Theorem [5, 48]."

TODO Causal+ [@lloyd2011don]
"able to leverage the existence of multiple replicas to distribute the load of read requests"

This more relaxing consistency models in general violate crucial correctness properties, compared with strong consistency. A compromise is to allow multiple consistency levels to coexist in the data store, which can be achieved depending on the use case and constraints of the system. An example is to combine both strong and causal consistency for applications with geographically replicated data in distributed data centers [@bravo2021unistore].

**Weak Consistency** - 
TODO do we need to describe this in the scope of dist sys? This is in the scope of processors(concurrency)! 

- Data-centric
"As its name indicates, weak consistency offers the lowest possible ordering guarantee, since it allows data to be written across multiple nodes and always returns the version that the system first finds. This means that there is no guarantee that the system will eventually become consistent."

Only writes that are explicitly synchronized are consistent. Everything else in between is unordered (but at least the same set of operations)

Needs programmers to explicitly synchronize operations, like it is possible with the well-known synchronize keyword in Java

A protocol is said to support weak consistency if:

"All accesses to synchronization variables are seen by all processes (or nodes, processors) in the same order (sequentially) - these are synchronization operations. Accesses to critical sections are seen sequentially.
All other accesses may be seen in different order on different processes (or nodes, processors).
The set of both read and write operations in between different synchronization operations is the same in each process."

A trivial implementation of this would be replicas kept locally in the client that are not synchronized.




\todo{Table summarizing the consistency models, showcasing differences in availability, reliability and safety}


#### The CAP Theorem

Why can't we always have strong consistency? The CAP theorem (CAP stands for Consistency, Availability and Partition Tolerance), also known as Brewer's Theorem, named after its original author Eric Brewer [@brewer2000towards], asserts that the requirements for strong consistency and high availability cannot be met at the same time and serves as a simplifying explanatory model for consistency decisions that depend on the requirements for the availability of the distributed system in question [@gilbert2002brewer].

TODO draw and show the diagram

TODO also this https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSLf5rylrAHnisE9SVugRCqKIdJu63fiiWkFXPIydsz_8HmSOp-Afwe0IySCaR_ow3GlUM&usqp=CAU

Traditional, non-distributed relational database management systems (RDBMS) support both consistency and availability (as there is either one single node available or none), while they do not support partition tolerance, which is mandatory for distributed databases. 
\todo{Source for RDBMS, clustering etc}

In general, partition tolerance should always be guaranteed in distributed systems (as described above in the [Types of Possible Faults](#sec:possible-faults)). Therefore, the tradeoff is to be made between consistency and availability in the case of a network partition. Eric Brewer revisited his thoughts on the theorem, stating that tradeoffs are possible for all three dimensions of the theorem: by explicitly handling partitions, both consistency and availability can be optimized [@brewer2012cap].

The theorem is often interpreted as a proof that eventually consistent databases have better availability properties than strongly consistent databases. There is sound critic that in practice this is only true under certain circumstances: the reasoning for consistency tradeoffs in practical systems should be made more carefully and strong consistency can be reached in more real-world applications then the CAP theorem would allow [@kleppmann2015critique].

#### The CALM Theorem

The CALM theorem (CALM stands for...) is a critic on the CAP theorem that says...

[@hellerstein2019keeping]


### Partitioning and Sharding {#sec:partitioning}

- https://dimosr.github.io/partitioning-and-replication/ 
- https://dev.mysql.com/doc/refman/5.7/en/replication-features-partitioning.html

- Performance is critical for many applications, especially for distributed ones
- sharding (also known as horizontal partitioning) helps to provide high availability, fault-tolerance, and scalability to large databases in the cloud [bagui2015database]
- Partitioning and sharding are methods to ensure throughput by providing horizontal scalability 
  - Explain horizontal scal.

- Various Partitioning strategies 
  - Reference hashing, Consistent hashing (Cassandra strategy), time split strategies etc. here
- Strategies can also support geograhical scalability, see following subchapter

TODO obsidian notes here

TODO compare with replication, show later how multi-raft supports both

TODO diagrams, also my own ones from PPT

### Geo-Replication {#sec:geo-replication}

The replication techniques discussed so far in this work describe the replication of data within a single cluster to improve the dependability of the system. To improve performance when accessing the system and data from different geographical regions and to hedge against geographical limited disasters, data can be further replicated across clusters using _data mirroring_ techniques, which is referenced as _cross-cluster replication_.

TODO more from https://kafka.apache.org/documentation/#georeplication
- Also for feeding edge cluster data into one central cloud cluster
- In general, this is solved by simply cloning the full data in the background, either hot (while the primary cluster is still running) or cold (when the cluster is shutdown, to produce a reliable cold backup)

For geo-replication, a client-centric consistency model is sufficient as not all data 

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

### Cost of Replication

As far as we now discussed the benefits of replication for failure management, for Performance enhancements by reducing latency in edge computing and geographically distributed networks, and to increase dependability in general, we need to discuss the _Cost of Replication_. Dependent on the level of fault-tolerance, the consistency decisions and the selected protocol itself, the performance, especially for write operations, can decrease dramatically. We briefly discuss appropriate strategies for a performance-tradeoff mitigation in this case.

Modification on one replica triggers modification on all other replicas --> messaging and acknowledgment of all operations between replicas degrades performane

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

There are examples that anecdotally show that high availability and high throughput are possible even under strong consistency constraints: AWS S3 introduced strong consistency for their file operations[^s3-eventual] recently in 2021 [@amazon2021s3consistency]. They even claim to deliver strong consistency "without changes to performance or availability", compared to the earlier eventual consistency.

[^s3-eventual]: Bucket configuration operations are still eventual consistent in AWS S3 at the time of this writing.

#### Partial Replication

Similar to the considerations of client-centric consistency models... in many systems, not all clients need to access the same data. If data portions can be identified that are only accesses by some clients, they don't need to be consistently shared between all replicas...

TODO [@shen2015causal]

###  Error Correcting Codes

TODO only a short excursion

### Replication Protocols

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

Certain different variations of how to understand a consensus protocol appear in the literature. They differ in the assumed properties of the messaging system, in the type of errors allowed to the processes, and in the notion of what a solution is. Most notable is the differentiation between _consensus protocols for asynchronous and synchronous message-passing systems_. Fischer et al. have proven in the famous _FLP impossibility result_ (called after their authors) that a deterministic consensus algorithm for achieving consensus in a fully asynchronous distributed system is impossible if even a single node crashes in a fail-stop manner [@fischer1985impossibility]. They investigated deterministic protocols that always terminate within a finite number of steps and showed that "every protocol for this problem has the possibility of nontermination, even with only one faulty process." This is based on the failure detection problem discussed earlier in the [subsection on Types of Possible Faults](#sec:possible-faults): in such a system, a crashed process cannot be distinguished from a very slow one.

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

Popular examples for distributed consennsus protocols are [Paxos](#sec:paxos) and [Raft](#sec:raft), the latter being the focus of this work.

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

TODO image from https://arxiv.org/pdf/1904.04098.pdf

TODO cite the slides for the State-Machine Approach for fault-tolerance https://www.inf.ufpr.br/aldri/disc/slides/SD712_lect13.pdf

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

#### Optimistic Replication

There are many use cases where the cost of replication is too high when synchronization between nodes in a replica set is required, especially for real-time live collaboration applications or applications that cannot be guaranteed to be online all the time (such as mobile apps). In these cases, the consistency requirements should be attenuated to eventual consistency so that the application can be designed in a non-blocking manner. One approach to solving problems of this scope is _optimistic replication_, as described by Shapiro et al. [@shapiro2005optimistic]. In optimistic replication, any client managing a replicated state is allowed to read or update the local replica at any time.

\begin{figure}[h]
  \centering
  \includegraphics[width=0.8\textwidth]{images/crdts-optimistic-replication.pdf}
  \caption{Schematic communication model between the nodes in optimistic replication for a single operation. A write is acknowledged immediately and non-blocking, without synchronization with other nodes. It is then eventually propagated to other nodes. In the event of a write conflict, the conflict is automatically resolved in the background using either a decentralized protocol or a centralized entity. Once the conflict is resolved, the actual agreed value is written back to the node's internal state and acknowledged.}
  \label{fig:crdts-optimistic-replication}
\end{figure}

Such local updates are only tentative and their final outcome could change, as they may conflict with an update from another client. In optimistic replication, conflicts should not be visible to the user (since conflict resolution requiring human intervention would compromise consistency, which is undesirable in the intended use cases), so they must be resolved in the background, either by a decentralized protocol or a central server instance. The replicas are allowed to differ in the meantime, but they are expected to eventually become consistent. Figure \ref{fig:crdts-optimistic-replication} illustrates this behavior. Note that it only illustrates it for a single operation. In case of multiple operations, reordering the operations is also part of the conflict resolution, depending on the protocol. 

\todo{Describe steps of OR: server/decentralized protocol defines order of events...}

\todo{Possible to use CRDTs for chronicledb TAB+ tree?}

An example of optimistic replication for strong eventual consistency are _conflict-free replicated data types_ (CRDT). CRDTs were introduced by Shapiro et al. and describe data types with a data schema and operations on it that will be executed in replicated environments to ensure that objects always converge to a final state that is consistent across replicas [@shapiro2009crdts]. CRDTs ensure this by requiring that all operations must be designed to be conflict-free, since CRDTs are intended for use in decentralized architectures where operation reordering is difficult to achieve. Therefore, any operation must be both _idempotent_ (the same operation applied multiple times will result in the same outcome) and _commutative_ (the order of operations does not influence the result of the chained operations), which also requires them to be free of side effects [^secro].

CRDTs take the consistency problem onto the level of data structures. They make use of optimistic replication to allow for such operations without the need of replicas to coordinate with each other (often allowing updates to be executed even offline) by relying on merge strategies to resolve consistency issues and conflicts, once the replicas synchronize again. They are designed in a way that the overall state resolves automatically, becoming eventually consistent. Due to the nature of the operations to be idempotent, CRDTs can only be used for a specific set of applications, such as distributed in-memory key-value stores like Redis, which is commonly used for caching [@redis2022crdts]. Another common set of use cases are realtime collaborative systems, such as live document editing (relying on suitable data structures to hold the characters with good time complexity properties for both random inserts and reads, such as an _AVL tree_ [@adelsonvelskii1963algorithm]) [@shapiro2009commutative]. CRDTs are not suitable for append-only logs, streams or event stores, as designing an operation to append a record or event with idempotency and commutativeness is too expensive.

[^secro]: Recent research is trying to find such a replicated data type that does not require operations to be commutative. One approach describes _Strong Eventually Consistent Replicated Objects_ (SECROs), aiming to build such a data type with the same dependability properties as CRDTs [@de2019generic]. The authors achieve this by ensuring a total order of operations across all replicas, but still without synchronisation between the replicas: they show that it is possible to order the operations asynchronously.

CRDTs are not the only way to achieve optimistic replication. Before CRDTs, _operational transformations_ (OT) were the default approach for solving these kinds of problems, at least in academia. Due to their complexity and the difficulty to implement them and also to write formal proofs for different sets of operations needed in practical applications [@li2010admissibility], they were often replaced by alternative approaches like CRDTs, so we will not discuss them in detail in this work.

\todo{Grammar checking}
In theory, CRDTs are designed for decentralized systems in which there is no central authority to decide the end state. In practice, there is oftentimes a central instance, such as in SaaS offerings on the web. Decentralized conflict resolution is no longer a requirement for those systems, so CRDTs could be too heavy-weight. Moving the conflict resolution to a central instance (which actually could be a cluster of nodes with stronger consistency guarentees) reduces complexity in the implementation of optimistic replication. This is actually the case in multiplayer video games—especially massive multiplayer online games (MMOGs). They introduce a class of problems that need techniques to synchronize at least a partial state between a lot of clients. Strong consistency would cause this games to experience bad performance drops, since a game's client wouldn't be able to continue until it's state is consistent across all subscribers to that state, so the way to go is eventual consistency. One can learn a lot from these approaches and adopt it to other real-time applications, such as Figma did for their collaborative design tool, comprehensively described in a blog post [@figma2019multiplayer].

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


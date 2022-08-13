## Event Stores and Temporal Databases {#sec:event-stores}

\epigraph{Database: the information you lose when your memory crashes.}{--- \textup{Dave Barry}}

Event stores are a type of database optimized for storing and managing immutable event streams. They where developed out of _temporal databases_ [@jensen2009temporal].

Temporal databases have been the subject of research since the early 1980s [@bolour1982role; @snodgrass1992temporal]. Temporal tables for relational databases where first proposed in 2011 with the _SQL:2011_ proposal, including _time validity period_ tables. Instead of overwriting a table column on an update, those tables will create snapshots of the last column state on each update, annotated with its period of validity, to preserve history.

Distributed databases became important with the advent of NoSQL databases like MongoDB, which—at least in their early years—where often only eventually consistent in order to ensure availability, partition tolerance, and performance. The discussions around distributed databases that drop ACID properties for the sake of performance started in 2009[^no-sql]. Around the same time, distributed temporal and time-series databases where discussed. They gained popularity in industries and businesses with the rise of IoT and Big Data and the growing demand for organizations to perform present-day and real-time data processing, as well as event stores with event-driven software architecture paradigms.

[^no-sql]: https://web.archive.org/web/20110716174012/http://blog.sym-link.com/2009/05/12/nosql_2009.html

Distributed event stores generally manage append-only log structures with immutable events. This shifts the discussion of ACID properties and transactional consistency to a discussion of immutable and ordered sequences of state changes. Relational, but also NoSQL databases, generally manage intermediate and final states of real-world objects, while event stores manages the events that describe the state changes. Thanks to the append-only property, event stores are in general tremendously faster than relational databases, and reduces the need for transactions when combined with methods such as complex event processing (CEP). Note that this is not always comparable, because to describe the same state as in a relational database, you need to write and read multiple events.

We present a selection of distributed temporal databases, event stores and time-series databases in subchapter [@sec:previous-work].

### What are Events? {#sec:events}

An event[^event-probability] is a notification about an observation from the real world. An _event stream_ is a temporally ordered sequence of events and thus a special form of a data stream. An event belongs to a certain _domain_, which is expressed by the _event schema_. Typically, an event stream contains only events of a single schema [@etzion2009eventstream]. Event streams can have multiple order relations, such as the insertion time and the event time (see section [@sec:event-time]), while supporting multiple orders is generally not trivial in implementation.

[^event-probability]: Not to be confused with the definition of an event in probability theory.

We define events and related concepts using the following notation, based on some of the definitions of Koerber [@koerber2021accelerating] and Seidemann et al. [@seidemann2019chronicledb]):

\begin{definition}[Event]\label{def:event}
An event $e_t$ is a tuple of several non-temporal, primitive attributes $(a_1, \dots, a_n)$ that resembles a notification about observations of facts and state changes from the real world at a timestamp $t$\footnote{There can be two events at the same timestamp $t$. In general, the subscript of $e$ denotes the index of the event in an event stream, based on event time ordering. We ignore this case in this work to simplify the following discussions and because it is not needed for understanding. Unless otherwise specified, $e_t$ denotes an event that happened at timestamp $t$ in event time.}. An event is immutable and persisted in an event stream.
\end{definition}

\begin{definition}[Event Stream]
An event stream $E$ is a potentially unbounded sequence of events $\langle e_1, e_2, \dots \rangle$ that is totally ordered by event time. We call the domain of event streams $\mathcal{E}$.
\end{definition}

\begin{definition}[Time Window]
$E_{[t_i,t_j)}$ refers to a time window of an event stream, which is a slice of the stream containing all the events with timestamps in the time interval $[t_i,t_j)$ (excluding events at timestamp $t_j$).

A time windowing function $W_d$ applied to an event stream $E$ results in a non-overlapping sequence of such time windows. The time windowing function for the duration $d$ is defined as 
\begin{equation*}
    \begin{split}
        W_d(E) = \langle \\
        & E_{[t_1,t_1 + d)}, \\
        & E_{[t_1 + d,t_1 + 2d)},\\ \dots \rangle
    \end{split}
\end{equation*}

Note that we use a simplified definition of windowing here as we only care about tumbling time windows, which means that windows don't overlap.
\end{definition}

While _temporal facts_ describe real-world facts (intermediate states) that are valid in a given period of time, an event often describes a occurence that leads to a change of such a fact. For example:

\begin{table}[H]
    \caption{A temporal table for temperature measurements}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r r >{\texttt}l >{\texttt}l} 
        \toprule
        \thead{Sensor} & \thead{Temperature} & \thead{$t_s$} & \thead{$t_e$} \\
        1 & 34.7 & \makecell{\texttt{2022-02-13T12:31:42}} & \makecell{\texttt{2022-02-13T12:36:11}}  \\
        1 & 36.1 & \makecell{\texttt{2022-02-13T12:36:11}}  & \makecell{\texttt{2022-02-13T12:38:52}}  \\
        1 & 35.2 & \makecell{\texttt{2022-02-13T12:38:52}}  & \makecell{\texttt{2022-02-13T12:41:12}}  \\
        \bottomrule
    \end{tabularx}
    \label{table:temporal-data}
\end{table}

\begin{table}[H]
    \caption{An event stream for temperature measurements}
    \centering
    \def\arraystretch{1.5}
    \begin{tabular}{>{\bfseries}r r c} 
        \toprule
        \thead{Sensor} & \thead{Temperature} & \thead{$t$} \\
        1 & 34.7 & \makecell{\texttt{2022-02-13T12:31:42}} \\
        1 & 36.1 & \makecell{\texttt{2022-02-13T12:36:11}} \\
        1 & 35.2 & \makecell{\texttt{2022-02-13T12:38:52}} \\
        \bottomrule
    \end{tabular}
    \label{table:event-data}
\end{table}

In table \ref{table:temporal-data}, one measurement of temperature is completed once the next is started, since the end of the validity period is known only with the start of the next measurement. In table \ref{table:event-data}, the same measurements are described by the event immediately at the time of their occurence only. The one table can be derived from the other. In continuous observations, the latest row of the temporal table describes the latest known state of the observed entity. The next example is even more figurative:

\begin{table}[H]
    \caption{A relational table for a person}
    \centering
    \def\arraystretch{1.5}
    \begin{tabular}{>{\bfseries}r r c c}
        \toprule
        \thead{Name} & \thead{Lives in} & \thead{Born on} & \thead{Works at} \\
        John & Berlin & 1985-03-23 & ACME Corp \\
        \bottomrule
    \end{tabular}
    \label{table:relational-data-person}
\end{table}

\begin{table}[H]
    \caption{A temporal table for a person's residence}
    \centering
    \def\arraystretch{1.5}
    \begin{tabular}{>{\bfseries}r r c c}
        \toprule
        \thead{Person} & \thead{Lives in} & \thead{$t_s$} & \thead{$t_e$} \\
        John & New York & \makecell{\texttt{2015-05-01}} & \makecell{\texttt{2017-03-01}}  \\
        John & Ontario & \makecell{\texttt{2017-03-01}} & \makecell{\texttt{2020-06-01}}  \\
        John & Berlin & \makecell{\texttt{2020-06-01}} & \makecell{$\infty$}  \\
        \bottomrule
    \end{tabular}
    \label{table:temporal-data-person}
\end{table}

\begin{table}[H]
    \caption{An event stream for a person's workplace}
    \centering
    \def\arraystretch{1.5}
    \begin{tabular}{>{\bfseries}r r c} 
        \toprule
        \multicolumn{3}{c}{\thead{person.job.applied}} \\
        \midrule
        \thead{Person} & \thead{City} & \thead{$t$} \\
        John & New York & \makecell{\texttt{2015-05-01}} \\
        John & Ontario & \makecell{\texttt{2017-03-01}} \\
        John & Berlin & \makecell{\texttt{2020-06-01}} \\
        \bottomrule
    \end{tabular}
    \label{table:event-data-person}
\end{table}

While the relational table in \ref{table:relational-data-person} only shows the latest known state for that person, we can model both `Lives in` and `Works at` using temporal tables or event streams.

### Time in Event Stores {#sec:event-time}

Temporal databases often come with multiple axes of time. Common axes are: 

- **Event time**: The time the event occured in the original application (also referred to as _application time_ or _valid time_),
- **Arrival time**: The time when the event arrived at the event store (also referred to as _transaction time_),
- **Insertion time**: The time when the event was stored and processed in the event store (also referred to as _decision time_).

In most systems, there is no distinction between arrival time and insertion time. On the event time axis, some systems—especially temporal databases—not only contain one timestamp per event, but two to demark the start and end of the _temporal validity_ period $[t_s, t_e)$ of this event. In databases storing temporal facts instead of events, this is used to describe the period of time that this temporal fact was valid.

To ensure strong consistency without additional out-of-order handling on queries, the ordering of the events based on arrival time and insertion time must be the same. We will elaborate this further in subchapter [@sec:system-design].

<!-- https://developer.confluent.io/patterns/stream-processing/wallclock-time/ -->

##### High-Resolution Timestamps.

In practice, many applications and sensors provide a very low time resolution of milliseconds, microseconds or sometimes even nanoseconds (such as in experimental setups of particle physics). In addition, consecutive events may carry equal timestamps. The ordering of events is therefore at least non-decreasing. Event stores need to support this property without violating consistency. Due to multiple sources of events with slightly offset clocks and especially with very high throughput, it is a challenge to synchronize event time. General computer operating system are not able to resolve event time at submicrosecond level, especially as CPU multi-tasking/scheduling is not able to provide this resolution. This requires solutions including optimizations on the network level to have such high-resolution timestamps, such as the _Precision Time Protocol_ (PTP) [@watt2015ptp].

### Differentation of Event Stores and Time Series Databases

An event store contains and manages events in multiple _event streams_, while a time series database stores and manages _time series_.

A time series consists of multiple _data points_ on a time axis. A data point is a record of a _measurement_ of an entity. Usually, data points on time series are uniformally distributed on the time axis, in contrast to events, that usually occur in unpredictable patterns [@liu2009encyclopedia].

Time series databases support both temporal correlation and compression, but typically do not provide query support for operations such as pattern matching and complex event processing.

\begin{table}[H]
    \caption{A time series for temperature measurements}
    \centering
    \def\arraystretch{1.5}
    \begin{tabular}{>{\bfseries}r l l r} 
        \toprule
        \thead{$t$} & \thead{measurement} & \thead{field} & \thead{value} \\
        \makecell{\texttt{2022-02-13T12:31:42}} & temperature & sensor:1 & 34.7 \\
        \makecell{\texttt{2022-02-13T12:32:41}} & temperature & sensor:2 & 32.7 \\
        \makecell{\texttt{2022-02-13T12:32:41}} & temperature & sensor:1 & 36.1 \\
        \makecell{\texttt{2022-02-13T12:33:392}} & temperature & sensor:1 & 34.9  \\
        \bottomrule
    \end{tabular}
    \label{table:time-series}
\end{table}

In table \ref{table:time-series}, a typical time series table is shown. It represents the same observations as the event stream in table \ref{table:event-data}, but its structure is different: while an event stream contains ordered events matching a single schema per stream with any arbitrary payload, a time series row represents a measurement at a specific time. A measurement has a specific type, a field of the measurement and the actual measurement value for that field. Besides that, it can contain any number of additional labels or tags (which are further columns). As a result, time series typically contain a single measurement per data point, and multiple different measurements per time series, while an event can contain multiple measurements that happen at the time of the event. A time series measuring both temperature and humidity would be represented by two event streams, one for each measurement, since changes in temperature and humidity may not happen at the same time. While a measurement is triggered by a measuring device, so generally producing equidistant data points, independently of an actual change of the subject of measurement, an event is triggered by the change of the subject itself, which could occur at random intervals. This also means that in time series, the measuring system is responsible for generating the data point, and in event streams, the system where the event occured produces the event. Both time series and event data are immutable. We listed all the differences and similarities in table \ref{table:event-stores-vs-time-series}.

\begin{table}[H]
    \caption{Differences between Event Stores and Time-Series Databases}
    \centering
    \def\arraystretch{1.5}
    \begin{tabularx}{\textwidth}{>{\bfseries}r | X X} 
        \toprule
        & \thead{Event Stores} & \thead{Time Series DBs} \\
        \midrule
        Store Format & Event Stream & Time Series \\
        Granularity & Events & Measurements \\
        Record distribution & Random & Uniform \\
        Mutability & Immutable & Immutable \\
        Record producer & Measured system & Measuring system \\
        Scope & One schema per stream & Multiple measurement types per series \\
        Record Payload & Arbitrary & One measurement + labels \\
        \bottomrule
    \end{tabularx}
    \label{table:event-stores-vs-time-series}
\end{table}

### Relationship to Data Stream Management Systems

A _data stream management system_ (DSMS) extends a database management system by managing continuous streams of data, and _continuous queries_ on those streams [@ahmad2009dsms]. This means that instead of storing data first and then allowing systems to run ad-hoc queries, data directly runs in streams through a stream processor, where continuos queries are executed on the streams in real-time. Persisting the stream data is optional in such systems, and can occur either in embedded storage or an external database. The data must not neccessarily be events, it can be any kind of ordered data stream, but it appears naturally to be events in many cases. Unlike traditional data management systems, and due to the nature of stream applications, DSMSs must respond to incoming data and deliver results to consumers frequently and almost immediately. Managing unbounded streams with limited memory and CPU is one of the biggest challenges for a data stream management system. A DSMS is not restricted to continuous queries, it can also allow for ad-hoc queries by storing streams permanently. We illustrated the difference between a DSMS and traditional DBMS in figure \ref{fig:dsms}, based on [@ahmad2009dsms].

\begin{figure}[H]
  \centering
  \includegraphics[width=1\textwidth]{images/dsms.pdf}
  \caption[DSMS vs. traditional storage architecture]{Data stream management system vs. traditional storage architecture. a) In traditional storage architectures, data is directly persisted (often with transaction handling), before it is available for ad-hoc queries. b) In data stream management systems, data comes in as continuous and unbounded streams, and is immediately processed by a continuous query processor. Afterwards or in parallel, it is optionally persisted in the storage.}
  \label{fig:dsms}
\end{figure}

Data appearing out-of-order must also be taken into account by a DSMS and may be tolerated in controlled bounds. DSMS have a broader and generic scope, and often consist of multiple systems and services. For instance, they could cover complex event processing (CEP) systems: a DSMS allows for any arbitrary real-time data processing, while CEP systems are especially build for event (pattern) detection. An event store that supports continuous queries in real-time can can be considered as a part of a DSMS, responsible for managing, serving and persisting continuous streams.

### Use Cases and Challenges {#sec:event-store-use-cases}

Event stores, DSMS and related database systems support a wide range of use cases. For example, such systems allow to measure and track software and hardware metrics, including

- System health monitoring,
- Network analysis,
- Fraud detection,
- As well as usage and behavioral data analysis.

These systems are also used in measuring sensor data and events, including

- Personal health data (e.g., from fitness trackers),
- Medical data,
- Physics and biology experiments,
- Video data (e.g., from surveillance cameras),
- IoT,
- Industry 4.0,
- Smart grids,
- And smart cities.

From the system design perspective, there are the following applications:

- Event-sourcing and event-driven systems,
- Message brokers,
- Complex event processing (CEP),
- Stream processing,
- Process mining,
- And machine learning.

Many of this cases require fault-tolerance, high availabilty, very high write throughput and low latency for queries, while scaling with huge numbers of sensors and devices. This creates a strong need for distributed event stores. We will encounter some of these use cases again throughout this work.

#### Replicated Event Stores as a Replication Service

Similar to ZooKeeper, replicated event stores can be deployed as a replication service themselves, running next to a distributed system and offering the service to store and replicate the commands of this system. From this point of view, an event store turns out to be a replicated log—with some powerful indexing features. Event-sourced systems actually rely on the event store to hold the (replicated) source of truth for the overall system and individual application state.

However, this approach is not recommend for all use cases, as it adds additional unwanted latency by enforcing back-and-forth between the services, which is also a reason why Kafka decided to replace ZooKeeper by a quorum-based, embedded replication protocol.

\pagebreak

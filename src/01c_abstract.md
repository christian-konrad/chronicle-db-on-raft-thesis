\pagenumbering{roman}

\chapter*{Abstract}

\begin{otherlanguage*}{american}

ChronicleDB is a high-performance event store designed for the age of IoT and data-intensive event stream processing. It runs embedded in other applications on low-cost hardware, such as sensors, and standalone on individual machines, significantly outperforming the competition for both writes and queries. Unfortunately, ChronicleDB cannot be deployed on a distributed cluster with high availability, scalability and fault-tolerance, since it does not provide a data replication layer.

To bridge the gap and enable ChronicleDB to run in full edge-to-cloud networks, we present an approach for a distributed ChronicleDB in this work. To accomplish this, we collected requirements for the various use cases of distributed data stream management systems. We investigated dependability, fault-tolerance, and consistency models under the conditions of the CAP and PACELC theorems, and compared state-of-the-art implementations in the industry. We finally chose the Raft consensus algorithm as it satisfies these requirements. We implemented a prototypical replication and partitioning layer for ChronicleDB based on our decisions, and evaluated it in terms of the trade-offs to be made.

Our evaluation has shown that we can achieve fault-tolerance, availability, and scalability, but at the cost of lower throughput for a single event stream. We also pointed out potential optimizations that can reduce replication costs and overall write and read latency, which can be helpful in developing a production-ready distributed ChronicleDB.

\end{otherlanguage*}

\pagebreak

\chapter*{Zusammenfassung}

\begin{otherlanguage*}{ngerman}

ChronicleDB ist ein hochleistungsfähiger Ereignisspeicher, der für das Zeitalter von IoT und datenintensiver Ereignisstromverarbeitung entwickelt wurde. ChronicleDB läuft eingebettet in anderen Anwendungen auf kostengünstiger Hardware, wie z. B. Sensoren, und eigenständig auf einzelnen Maschinen und übertrifft die Leistung der Konkurrenz sowohl bei Schreibvorgängen als auch bei Abfragen erheblich. Leider kann ChronicleDB nicht in einem verteilten Cluster mit hoher Verfügbarkeit, Skalierbarkeit und Fehlertoleranz eingesetzt werden, da es keine Datenreplikationsschicht bietet. 

Um diese Lücke zu schließen und ChronicleDB den Betrieb in vollständigen Edge-to-Cloud-Netzwerken zu ermöglichen, stellen wir in dieser Arbeit einen Ansatz für einen verteilten ChronicleDB-Ereignisspeicher vor. Um dies zu erreichen, haben wir die Anforderungen für die verschiedenen Anwendungsfälle von verteilten Datenstrommanagementsystemen gesammelt. Wir untersuchten die Begriffe Zuverlässigkeit, Fehlertoleranz und Konsistenzmodelle unter den Bedingungen der CAP- und PACELC-Theoreme und verglichen die neuesten Implementierungen von verteilten Ereignisspeichern, Zeitreihendatenbanken und vergleichbaren Systemen. Wir entschieden uns schließlich für den Consensus-Algorithmus Raft, da er diese Anforderungen erfüllt. Auf der Grundlage unserer Entscheidungen haben wir eine prototypische Replikations- und Partitionierungsschicht für ChronicleDB implementiert und sie im Hinblick auf die zu treffenden Kompromisse bewertet.

Unsere Evaluierung hat gezeigt, dass wir Fehlertoleranz, Verfügbarkeit und Skalierbarkeit erreichen können, allerdings auf Kosten eines geringeren Durchsatzes für einen einzelnen Ereignisstrom. Wir haben auch potenzielle Optimierungen aufgezeigt, die die Replikationskosten und die gesamte Schreib- und Leselatenz verringern können, was bei der Entwicklung einer produktionsreifen verteilten ChronicleDB hilfreich sein kann.

\end{otherlanguage*}

\pagebreak

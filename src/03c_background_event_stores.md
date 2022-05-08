## Event Stores and Time Series Databases {#sec:event-stores}

\epigraph{Database: the information you lose when your memory crashes.}{--- \textup{Dave Barry}}}

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

- Append-Only Logs
- Extremely High Throughput
- Schema-Safety
- Streams, Topics, Queues...
- Transactional vs. Fire-and-Forget (?)
- etc.

### Use Cases and Challenges

#### Differentation of Event Stores and Time Series Databases

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

TODO cite papers from
https://epb.bibl.th-koeln.de/frontdoor/deliver/index/docId/1395/file/Vergleich-Time-Series-Databases-Event-Stores.pdf 

Zwar ist es keine zwingende
Voraussetzung von Zeitreihendaten, vgl. auch Kapitel 2.1.2, jedoch ist es üblich, dass
Zeitreihendaten in regelmäßigen Zeitabständen gemessen und somit auch in regelmäßigen Zeitabständen in einem DBMS (TSDB) gespeichert werden.
59 Im Gegensatz
dazu finden Events i.d.R. nach unvorhersehbaren Mustern statt, wie auch das Beispiel
des User-Logins in einer Webanwendung verdeutlicht. Demzufolge erfolgt die Speicherung von Event Daten eher in unregelmäßigen Zeitabständen.60

#### Software and Hardware Metrics

- System Health Monitoring
- Network Analysis
- Fraud Detection
- UX Event Data

#### Sensor Data

- Medical
- Biological
- IoT
- Industry 4.0
- Smart Cities

#### Event-Sourcing and Event-Driven Systems

#### Messaging

- AMQP etc

#### Aggregation and Stream Processing

#### Out-Of-Order (Random Writes)

#### Differences between popular Event Stores

> Here or in Previous Work / Implementation?

### High Availability in Event Stores

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Strong Consistency in Event Stores

TODO describe transactions in event stores
TODO describe importance for online analytical processing
TODO describe ACID vs. SAGA pattern here if suitable

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Partitioning and Sharding of Streams

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

### Cross-Cluster Replication

Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

\pagebreak

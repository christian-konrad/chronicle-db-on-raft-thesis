## Methodology {#sec:methodology}

TODO put it here or before background?


<!-- See https://www.indeed.com/career-advice/career-development/how-to-write-a-methodology -->

<!-- Restate your thesis or research problem -->
- Identification of a consistency model suitable for an event store with a specific set of requirements and differentiation from other consistency models including a detailed discussion of the advantages and disadvantages of the models in question.
- Identification and justification of a replication protocol suitable for an event store that satisfies the characteristic requirements of the selected consistency model, including a differentiation from other replication techniques
- Investigation of the performance trade-off of an implementation of the chosen consensus-based, strong-consistent replication protocol in an event store to make it fault tolerant and horizontally scalable, compared with the expected trade-off.
<!-- Explain the approach you chose; Describe how you collected the data you used -->
- Embedded/Explorative Research Design: Conducting a qualitative, comparative analysis, then using a quantiative method (benchmarking) to confirm the results / that the expectations are met.
- Qualitative Untersuchung: Untersuchung via Evaluierung der Literatur zu Replikation (siehe vorhergehende Kapitel), Beschreibung der Anforderungen an die Implementierungen, Evaluierung und Vergleich bestehender Implementierungen (verschiedener populärer und akademisch relevanter Datenbanken und weiteren Stores), Auswahl eines geeigneten Konsistenzmodells und anschließend eines Replikationsprotokolls für diese Anforderungen, Implementierung des Protokolls in geeigneter Sprache und Framework und anschließende quantiative Untersuchung: Messung und Evaluation des Performance-Impacts anhand eines Benchmark-Vergleichs mit dem nicht-replizierten originalen Event Store
Ziel (Referenz auf Ausgangshypothese hier): Bestätigung der Implementierbarkeit einer Lösung für das Ausgangsproblem, Messung des Performance Impacts nach Abschätzung desselben (Ist es im Rahmen des erwarteten Performance Tradeoffs?) und Identifikation weiterer möglicher Schritte, um den Ausgangs- und weiteren Anforderungen zu genügen

<!-- Evaluate and justify the methodological choices you made
Describe the criteria you used in choosing your approach to your research. List any potential weaknesses in your methodology and present evidence supporting your choice. Include a brief evaluation of other methodology you might have chosen. -->
- TODO describe that the qualitative measurement against the benchmark is reliable (thus, results can be repeated by others by using the implementation) and valid (the results measure what they are supposed to measure)

<!-- Discuss any obstacles and their solutions
Outline any obstacles you encountered in your research and list how you overcame them. The problem-solving skills you present in this section strengthen the validity of your research with readers. -->

<!-- Cite all sources you used to determine your choice of methodology
The final section of your methodology references the sources you used when determining your overall methodology. This reinforces the validity of your research. -->

TODO is this a Comparative Analysis? Or if not, what is this at all? Any "Name" for this kind of implementation thesis?

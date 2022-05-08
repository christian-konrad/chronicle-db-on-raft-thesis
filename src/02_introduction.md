# Introduction {#sec:introduction}

\pagenumbering{gobble}
<!--- Use \shorthandoff{"} for german documents -->

<!---
You can use inline comments to organize yourself
-->

<!--- 
- [x] Start using boilerplate
- [ ] Your next TODO
-->

<!--- PARAGRAPH 1 - DESCRIBE IT HERE -->
At the foundation of data-intense applications are distributed event stores...


there are a lot of requirements to today's distributed systems: They must be able to ingest high throughputs, store a massive load of data (what is a massive load?) and respond almost immediately to user queries, while providing fault-tolerance, consistency and high availability at the same time.
 
 need to meet these requirements while serving millions of users...
 
For distributed event stores, those requirements are even higher (why?). With modern approaches to replication, it is possible to meet this requirements for both vertical and horizontal scalability...

TODO better, catchy introduction. See other work for inspiration

<!--
General motivation for your work, context and goals: 1-2 pages
Make sure to address the following: 
• Context: make sure to link where your work fits in
• Problem: gap in knowledge, too expensive, too slow, a deficiency, superseded technology
• Strategy: the way you will address the problem
-->

- TODO List the research question(s)
    - What are the unique characteristics of event stores? 
    - What are the use cases of event stores?
    - Which role do event stores play in distributed systems and what are the requirements on them in this context?
    - How to make event stores fault tolerant?
        - .. we want ... to be still operational... when one node fails...
    - What is replication, and why do we need it?
    - What are the different replication protocols and what are the differences between them?
    - Which protocol fits best and why? What are the advantages of this protocol and which are the disadvantages? 
        - What is the performance/throughput impact and what influences it? What are ways to improve it?
            - Are there any upper limits?
        - What do users have to do without?

- TODO Write a clear, interesting and specific hypothesis
    - With the utilization of a replication protocol, an event store becomes horizontally scalable, i.e., highly available, fault-tolerant, and more performant (as the number of streams increases).

- TODO thesis statement

See https://nmbu.instructure.com/courses/2280/pages/research-questions-hypotheses-and-thesis-statements?module_item_id=15684



\pagenumbering{arabic}
\setcounter{page}{2}

<!--- PARAGRAPH 2 - DESCRIBE YOUR STRUCTURE HERE -->
Lorem ipsum dolor sit amet, consetetur sadipscing elitr [Section 2](#sec:your-next-section), sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet.

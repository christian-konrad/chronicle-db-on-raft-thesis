# Protocol Decision and Implementation {#sec:implementation}

1-2 sentences on what this chapter is about. It starts looking at existing implementations of Raft and replication protocols in general for data-intense applications and how they differentiate. Then it compares other event stores with ChronicleDB and their approaches to replication. Afterwards, it describes the system design and architecture of the solution, (and finally it will reference the solution in particular (should I really show the code? Or just latex algos in the system design and omit details)).

<!--
In Chapter 3 we gave a formal description of the CS reconciliation protocol.
This allows the protocol to be conceptually understood and its properties
formally shown. However, it does not provide much information on how such
a protocol could be implemented in a real middleware. Therefore, we will in
this chapter describe two implementations that we have done: a simulated
environment based on J-Sim [53], and a component..
The reason that we chose to make two implementations is that they
provide different means of evaluation. The simulation environment allows
important system parameters to be changed, so that the behaviour of the
protocol can be investigated under different conditions. In the CORBA
implementation the protocol is tested on a real system, and provides more
realistic measurements results.
We chose J-Sim as the simulator platform since it is a component-based
simulation environment that provides event-based simulation. Moreover, it
is built with Java and uses TCL as glue code to control simulations. The
same Java code could therefore be used in both the simulation and later in
the CORBA middleware.
-->
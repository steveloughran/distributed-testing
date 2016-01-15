
# System Under Test

A core concept in Distributed System Testing is the *System Under Test* (SUT)

It is defined in [Ulrich99] as part of a series of definitions

1. A *distributed system* consists of a collection of components,
each of them realizing a sequential flow of execution.
Each component has an interface that defines its incoming and outgoing messages.

1. The System Under Test (SUT) is the executable implementation of
the distributed software system to be tested,

2. The purpose of testing, then is to verify that the behavior of the SUT matches that specified.

To test this implementation, the SUT must be extended with

* A test case and means of assessing the results of a test run,
* Points of control and observation (PCOs) and pure observation points (POs) between tester and the SUT
* Possibly test coordination procedures which coordinate the activities between different tester components.

Ullrich, Zimmerer and Chrobok-Diening also define both the distributed system as a Deterministic Finite State Machine, DFSM, 

>  A labeled transition system (LTS or machine for short) M is defined by
 a quadruple (S, A, , s0), where S is a finite set of states; A is a finite set of actions (the alphabet);
S A S is a transition relation; and s0 S is the initial state.

Actions which trigger state transitions may be external, *reactive*, or they may be internal, between components. Note that their model does not consider events triggered within components. When testing a system such as Hadoop, we lack that luxury.

They consider the problems of DST to be

1. non-determinism due to interleaving of concurrent actions of the SUT
2. non-determinism within a component of the SUT, and
3. non-determinism due to race conditions within the SUT.


We would add more problems.

1. The interference of components with each other. Multiple components running on the same host may unintentionally alter the behaviour of other components on the same system, possibly simply through excessive CPU or memory usage; perhaps by some behaviour triggering a failure condition elsewhere (for example: filling up a temporary directory with log messages, so using up disk space). Network use and traffic can mean that components across the entire cluster may cause interference.
2. The aggregate state of the system is actually defined by the entire state of the computers hosting the SUT and the networking, as well as the components themselves. That is, the SUT is the software as instantiated within a physical or virtualized cluster, rather than the software itself.
3. Time has to be considered. While messages between components is the communication mechanism, each host can be viewed as having its own (isolated) clock, moving forwards at approximately the same rate as those on other hosts. This is an integral requirement of the TCP/IP protocol, and used by layers above, from Zookeeper's ZAB protocol to the Hadoop IPC channel timeout and retry policies. This is a significant complication. However, as the primary concern is about scheduled events, one may view the OS scheduler as a component, whose (usually monotonically increasing) clock is used to trigger actions (thread scheduling, timeout notifications). Thus it too is a deterministic component within the SUT, a provider of action messages to other components on the same host.





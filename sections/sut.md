# System Under Test

A core concept in Distributed System Testing (DST_ is the *System Under Test* (SUT)

It is defined in [Ulrich99] as part of a series of definitions

1. A *distributed system* consists of a collection of components,
each of them realizing a sequential flow of execution.
Each component has an interface that defines its incoming and outgoing messages.

1. The *System Under Test* (SUT) is the executable implementation of
the distributed software system to be tested,

1. The purpose of testing, then is to verify that the behavior of the SUT matches that specified.

To test this implementation, the SUT must be extended with

* A test case and means of assessing the results of a test run,
* Points of control and observation (PCOs) and pure observation points (POs) between tester and the SUT
* Possibly test coordination procedures which coordinate the activities between different tester components.

Ullrich, Zimmerer and Chrobok-Diening also define both the distributed system as a Deterministic Finite State Machine, DFSM, 

>  A labeled transition system (LTS or machine for short) M is defined by
 a quadruple (S, A, , s0), where S is a finite set of states; A is a finite set of actions (the alphabet);
S A S is a transition relation; and s0 S is the initial state.

Actions which trigger state transitions may be external, *reactive*, or they may be internal, between components.
Note that their model does not consider events triggered within components.
When testing a system such as Hadoop, we lack that luxury.

They consider the problems of DST to be

1. Non-determinism due to interleaving of concurrent actions of the SUT
2. Non-determinism within a component of the SUT, and
3. Non-determinism due to race conditions within the SUT.


We would add more problems.

## State

The aggregate state of the system is actually defined by the entire state of the computers hosting
the SUT and the networking, as well as the components themselves.

That is, the SUT is the software as instantiated within a physical or virtualized cluster,
rather than the software itself.


Notably, multiple components running on the same host may unintentionally alter the
behaviour of other components on the same system, possibly simply through excessive CPU or
mmemory usage; perhaps by some behaviour triggering a failure condition elsewhere.
As examples: filling up a temporary directory with log messages, so using up disk space, simply
using up all file handles, or listening on a network port which another sevice also wishes
to listen to.

Network use and traffic can mean that components across the entire cluster may cause interference.


## Time

Time has to be considered. While messages between components is the communication mechanism,
each host can be viewed as having its own (isolated) clock, moving forwards at approximately
the same rate as those on other hosts.

This is an integral requirement of the TCP/IP protocol, and used by layers above, from Zookeeper's
ZAB protocol to the Hadoop IPC channel timeout and retry policies. This is a significant complication.
However, as the primary concern is about scheduled events, one may view the OS scheduler as a component,
whose (usually monotonically increasing) clock is used to trigger actions (thread scheduling, timeout notifications).
Thus it too is a deterministic component within the SUT, a provider of action messages to other components
on the same host.

Note also that some services require the callers's clocks to have absolute times close to that of
the services themselves. Kerberos is one example, Amazon AWS authentication another: the clocks
help defend against replay attacks.


## Failures.

We have to consider (at least) the non-Byzantine failures ---of components,
of hosts executing multiple components, and of entire racks, or the network infrastructure itself.

The ability to scale and support large workloads is essential.

While it's easy to say "correctness comes before scalability", some components within the big data/NoSQL space have deliberately chosen to relax consistentcy and durability guarantees in exchange for scalability and failure resilience. The correctness of such components becomes one of verifying that those guarantees are met.

Based on real-world data [], we can assume that such failures exhibit rack-locality.
Operations data from MSFT shows that ToR failures are the most common.
In a Hadoop cluster, failures at one layer of the stack may be exhibited in other layers of the system.
An HDFS failure, for example, may trigger failures in HBase, or Hive/Spark queries. These layers are required to be resilient to such failures -a requirement which tools such as Jepsen or the Netflix Chaos Monkey can help demonstrate. The latter is notable in that it is also run in production, so resulting in a host failure rate higher than  that which would normally be expected. Testing for failure resilience has essentially been extended beyond any pre-production test phase, into production itself.

The SUT is therefore the entire cluster: a set of hosts running our components, the network and its switches between the hosts, critical network services such as DNS,  KDC and NTP. The state of the SUT is therefore the internal state of all CPUs, their RAM, all persistent storage. That's not just of the hosts, strictly it should encompass the switches, the disk controllers, the NICs - maybe even the embedded HDD and SSD micro-controllers. Lamport [] has shown how a single computer executing a program can be considered on single DFSM, thus the entire SUT state is the aggregate state of all the DFSMs representing those computers.

This aggregate state is clearly impossible to model. Furthermore: it's impossible to observe completely. Some parts: the NICs, the disk controllers, cannot be directly inspected. The rest of the system may, individually be inspectable -but the act of inspection changes the state of the system. 
We have to conclude then, that

1. Attempts to completely evaluate the state of the SUT are impossible.
2. All that can be done is implement a set of PCOs to build a partial model of the SUT.
3. The act of monitoring may itself impact the state of the SUT. Implementing these PCOs without fundamentally changing the SUT is a challenge.
4. We can't look at a single application in isolation. When testing a single application, the PCOs attempting to evaluate the state of the entire cluster should still collect information.
5. When assessing resilience to failures, the layers above the failing application must be included in the test. That is: we must test their failure resilience by triggering failures in the layers beneath --even when those underlying layers are designed to recover from failures.
6. Load generation is an aspect of some test cases.


## Virtualization 

In a virtualized cluster, it is possible to snapshot the entire CPU, RAM and disk state of the virtualized hosts. This offers many opportunities --for replicating test conditions, or for analysing failures. It also aids fault injection.

Other key benefits are 

- The starting state of the VMs can be consistently replicated.
- Different sizes of test clusters can be instantiated.
- Chaos-monkey style test runners can trigger VM failures
- Some infrastructures allow network failures to be triggered.
 
Off-VM storage infrastructure can be a good destination for publishing service logs
Different CPU, RAM and storage options may be selected, providing the ability to benchmark on SSDs and large RAM systems.

There are some disadvantages

- If VMs are hosted on a pay-per-hour cloud infrastructure, costs usually mandate VM destruction once the test run has completed. 
- The networking is not representative of physical network infrastructures: bandwidth  throttling and latency being key points.
- Co-located VMs may interfere with each other.
- Public clouds are insecure: unless a VPN between VMs can be set up, the hosts need to be secured in ways which may interfere with operations or testing.
\footnote{ Of course, if an SUT is secure in a public cloud, then it is likely to be equally secure in production. However, it may make it harder to observe the SUT. }
- VM performance may vary between test runs.
- It's very expensive for full-scale test runs, such as multiple day petabyte-scale workloads.
The option to query the network infrastructure, such as recording sflow or openflow records from switches is lost,

## Points of Observation: moving beyond logs 

What do we have today?

- Logs.
- Information published for humans on web UIs.
- Information published for machines on REST APIs.
- Information retrievable from application specific IPC channels.
- Probes of the underlying state of the OS. Example: existence of files, processes.
- OS level system logs --unsually published to local filesystems; on windows they go to the event viewer. 
- Application logs

What are we neglecting?

- The metrics published by the instrumentation of applications intended for system monitoring and management
- OS metrics
- Intermediate snapshots of parts of the state of the SUT.
  For example: we could take a snapshot of the disk storage used by components.
- Switch network traffic records: sFlow and Netflow
- Information held or collected by any cloud infrastructure.
  That includes: operations made of storage services, network traffic information, 













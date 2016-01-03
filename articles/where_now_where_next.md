# Distributed System Testing: where now, where next? 

_Published, [May 17th, 2015](http://steveloughran.blogspot.co.uk/2015/05/distributed-system-testing-where-now.html)_

Confluent have announced they are looking for someone to work on an open source framework for distributed system testing.

I am really glad that they are sitting down to do this. Indeed, I've thought about sitting down to do it myself, the main reason I've been inactive there is "too much other stuff to do".

Distributed System Testing is the unspoken problem of Distributed Computing. In single-host applications, all you need to do is show that the application "works" on the target system, with its OS,  enviroment (timezone, locale, ...), installed dependencies and application configuration.

In modern distributed computing you need to show that the distributed application works across a set of machines, in the presence of failures.

Equally importantly: when your tests fail, you need the ability to determine why they failed.

I think there is much scope to improve here, as well as the fundamental problem: defining works in the context of distributed computing.

I should write a post on that in future. For now, my current stance is: we need stricter specification of desired observable behaviour and implementation details. While I have been doing some Formal Specification work within the Hadoop codebase, there's a lot more work to be done there —and I can't do it all myself.

Assuming that there is a good specification of behaviour, you can then go on to defining tests which observe the state of the system, within the configuration space of the system (now including multiple hosts and the network), during a time period in which failures occur. The observable state should continue to match the specification, and if not, you want get the logs to determine why not. Note here that "observed state" can be pretty broad, and includes

* Correct processing of requests
* The ability to serialize an incoming stream of requests from multiple clients (or at least, to not demonstrate non-serialized behaviour)
* Time to execute operations is one (performance),
* Ability to support the desired request rate (scalability)
* Persistence of state, where appropriate
* Reslience to failures of : dependent services, network, hosts, 
* Reporting of detected failure conditions to users and machines (operations needs)
* Ideally: ability to continue in the presence of byzantine failures. Or at least detect them and recover.
* Ability to interact with different versions of software (clients, servers, peers)
* Maybe: ability to interact with different implementations of the same protocol.

I've written some slides on this topic, way back in 2006, Distributed Testing with SmartFrog. There's even a sub-VGA video to go with it from the 2006 Google Test Automation Conference.

My stance there was

1. Tests themselves can be viewed as part of a larger distributed system
1. They can be deployed with your automated deployment tools, bonded to the deployed system via the configuration management infrastructure
1. You can use the existing unit test runners as a gateway to these tests, but reporting and logging needs to be improved.
1. Data analysis is a critical area to be worked on.

I didn't look at system failures, I don't think I was worry enough about that, showing we weren't deploying things at scale, and before cloud computing took failures mainstream. Nowadays nobody can avoid thinking about VM loss at the very least.

Given I did those slides nine years ago, have things improved? Not much, no

* Test runners are still all generating the Ant XML test reports written along with the matching XSLT transforms up by Stephane Balliez in 2000/2001
* Continuous Integration servers have got a lot better, but even Jenkins, wonderful as it is, presents results as if they were independent builds, rather than a matrix of (app, environment, time). We may get individual build happiness, but we don't get reports  by test, showing that Hadoop/TestATSIntegrationFailures is working intermittently on all debian systems -but has been reliable elsewhere. The data is all there, but the reporting isn't.
* Part of the problem is that they are still working with that XML format, one that, due to its use of XML attributes to summarise the run, buffers things in memory until the test test case finishes, then writes out the results. stdout and stderr may get reported -but only for the test client, and even then, there's no awareness of the structure of log messages
* Failure conditions aren't usually being explicitly generated. Sometimes they happen, but then it's complaints about the build or the target host being broken.
* Email reports from the CI tooling are pretty terse. You may get the "build broken, test XYZ with commits N1-N2", but again, you can get one per build, rather than a summary of overall system health.
* With a large dependent graph of applications (hello, Hadoop stack!), there's a lot of regression testing that needs to take place —and fault tracking when something downstream fails. 
* Those big system tests generate many, many logs, but they are often really hard to debug. If you haven't spent time with 3+ windows trying to sync up log events, you've not been doing test runs.
* In a VM/cloud world, those VMs are often gone by the time you get told there's a problem.
* Then there's the extended life test runs, the ones where we have to run things for a few days with Kerberos tokens set to expire hourly, while a set of clients generate realistic loads and random servers get restarted.

Things have got harder: bigger systems, more failure modes, a whole stack of applications —yet testing hasn't kept up.

In slider I did sit down to do something that would work within the constraints of the current test runner infrastructure yet still let us do functional tests against remote Hadoop clusters of variable size . Our functional test suite, [_funtests_](http://slider.incubator.apache.org/developing/functional_tests.html), uses Apache Bigtop's script launcher to start  Slider via its shell/py scripts. This tests those scripts on all test platforms (though it turns out, [not enough locales](https://issues.apache.org/jira/browse/SLIDER-876)), and forces us to have [a meaningful set of exit codes](http://slider.incubator.apache.org/docs/exitcodes.html) —enough to distinguish the desired failure conditions from unexpected ones. Those tests can deploy slider applications on secure/insecure clusters (I keep my VM configs [on github](https://github.com/steveloughran/clusterconfigs), for the curious), deploy test containers for basic operations, upgrade test, failure handling tests. For failure generation our IPC protocol includes messages to [kill a container](https://github.com/apache/incubator-slider/blob/develop/slider-core/src/main/proto/SliderClusterProtocol.proto#L112), and to have the AM [kill itself with a chosen exit code](https://github.com/apache/incubator-slider/blob/develop/slider-core/src/main/proto/SliderClusterProtocol.proto#L118).

For testing slider-deployed HBase and accumulo we go one step further. Slider deploys the application, and we run the normal application functional test suites with slider set up to generate failures.

How do we do that? With the [_Slider Integral Chaos Monkey_](http://slider.incubator.apache.org/developing/chaosmonkey.html). That's something which can run in the AM, and, at a configured interval, roll some virtual dice to see if the enabled failure events should be triggered: currently container and AM (we make sure the AM isn't chosen in the container kill monkey action, and have a startup delay to let the test runs settle in before starting to react).

Does it work? Yes. Which is good, because if things don't work, we've got the logs of all the machines in the cluster that ran slider to go through. Ideally, YARN-aggregated logs would suffice, but not if there's [something up between YARN and the OS](https://issues.apache.org/jira/browse/YARN-3561).

So: test runner I'm happy with. Remote deployment, failure injection, both structured and random. Launchable from my deskop and CI tooling; tests can be designed to scale. For testing rolling upgrades (Slider 0.80-incubating feature), we run the same client app while upgrading the system. Again: silence is golden.

Where I think much work needs to be done is what I've mentioned before: the reporting of problems and the tooling to determine why a test has failed.

We have the underlying infrastructure to stream logs out to things like HDFS or other services, there's nothing to stop us writing code to collect and aggregate those -with the recipient using the order of arrival to place an approximate time on events (not a perfect order, obviously, but better than log events with clocks that are wrong). We can collect those entire test run histories, along with as much environment information that we could grab and preserve. Junit: system properties. My ideal: VM snapshots & virtual network configs.

Then we'd go beyond XSLT reports of test runs and go to modern big data analysis tools. I'm going to propose here: Spark Why? so you can do local scripts, things in Jenkins & JUnit, and larger bulk operations. And for that get-your-hands-dirty test-debug festival, I can use a notebook like Apache Zepplin (incubating) can then be a no

we should be using our analysis tools for the automated analysis and reporting of test runs, the data science tooling for the debugging process.

Like I said, I'm not going to do all this. I will point to a lovely bit of code by Andrew Or @ databricks, [spark-test-failures](https://github.com/andrewor14/spark-test-failures). which gets Jenkins's JSON-formatted test run history, determines flaky tests and posts the results on google docs. That's just a hint of what is possible —yet it shows the path forwards.
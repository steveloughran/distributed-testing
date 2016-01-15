# Making use of the Metrics

_December, 2015_

I've been doing more scalatest work, as part of SPARK-7889, SPARK-1537. And, in SLIDER-82, anti-affine work placement across a YARN cluster, so that code can be spread across the cluster for reliability as well as performance.


On all these places, I've been exploring the notion of _metrics-based testing_. That is: your code uses metric counters as a way of exposing the observable state of the core classes, and then tests can query those metrics, either at the API level or via web views.

Take a test which was failing on December 4, 2015:

```
-- incomplete apps get refreshed *** FAILED ***
 The code passed to eventually never returned normally. Attempted 159 times over 20.019384 seconds. Last failure message: 2 did not equal 1 jobs not updated, server=
  History Server;
  provider =
  FsHistoryProvider: logdir=file:/Users/stevel/Projects/spark/core/target/tmp/spark-a2ae0d7a-ba8a-4bf7-95ff-a8d240a04b2f/,
  last scan time=1449247393000
  Cached application count =1}
 	FsApplicationHistoryInfo(lastUpdated = 1449247374000,
 	 ApplicationHistoryInfo(local-1449247373293,test,
 	 List(FsApplicationAttemptInfo(local-1449247373293.inprogress, test, local-1449247373293,
 	 ApplicationAttemptInfo(None,1449247372558,-1,1449247374000,stevel,false), 50882, 7))
  cache = ApplicationCache(refreshInterval=250, retainedApplications= 50); time= 1449247394099; entry count= 1
 ----
	local-1449247373293 -> UI org.apache.spark.ui.SparkUI@6124d9ce, data=Some(FsHistoryProviderUpdateState(2)), completed=false, loadTime=1449247373000 probeTime=1449247373740
 ----
 lookup.count = 645
 lookup.failure.count = 0
 eviction.count = 0
 load.count = 1
 update.probe.count = 638
 update.triggered.count = 0
 ----

  dir = ArraySeq(DeprecatedRawLocalFileStatus{path=file:/Users/stevel/Projects/spark/core/target/tmp/spark-a2ae0d7a-ba8a-4bf7-95ff-a8d240a04b2f/local-1449247373293.inprogress;
  isDirectory=false; length=50882; replication=1; blocksize=33554432;
  modification_time=1449247374000; access_time=0; owner=; group=;
  permission=rw-rw-rw-; isSymlink=false}).
  (HistoryServerSuite.scala:437)

```

Two points of interest here.

First: I'm using the classic `assert(2 === actual)` style of assertion, rather than the "the more expressive matchers DSL." of scalatest, which here would be `actual should be (2)`. Why? Because when an assert fails, I want to know the state of the system, rather than just 2 != 1 at line 429. So my "legacy" assert includes a better explanation, and diagnostics as built from the string values of objects in the system, objects which I've given meaningful `toString()` values for the sake of the tests. While I would like an elegant and declarative test language, given the choice between that and one which generates reports I can debug from a Jenkins run —I'll go for the other, no matter how ugly.

Second, that list of values at the bottom, from `"lookup-count = 653"` to `"update.triggered.count = 0"` are not just counters for the sake of test failures: _they are Codahale metric Counter instances_. As these are wrappers around atomic longs, you can just increment them after events, and check those values in asserts:

```java
assert(metrics.lookupCount.getCount > 1, s"lookup count too low in $metrics")
```

Nice and straightforward, no different from looking up an `AtomicLong` value in an object.

But, as they are metrics, _I can also check and assert on the state of remote services in the cluster_. That is, I can bring up a real spark history server instance in a real hadoop cluster, schedule real, work, have remote clients query the history server, _and make similar assertions over the state of those metrics_.

## By making the observable state of object instances real Metric values, I can extend their observability from unit tests to system tests. 

1. This makes assertions on the state of remote services a simple matter of `GET /service/metrics/${metric}` + parsing.
1. It ensures that the internal state of the system is visible for diagnostics of both test failures and production system problems. Here: are there too many evictions (==cache size too small)? Are too many update probes taking place (==probe window too small)? What about the lookup count (==metric of client load on the system). These are things which the ops team may be grateful for in the future, as now there's more information about what is going on.
1. It encourages developers such as myself to write those metrics early, at the unit test time, because we can get immediate tangible benefit from their presence. We don't need to wait until there's some production-side crisis and then rush to hack in some more metrics. Classes are instrumented from the outset. Indeed, in SPARK-11373 I'm actually implementing the metrics publishing in the Spark History server —something the SPARK-7889 code will be ready for.

I'm just starting to experiment with this metrics-first testing, idea —I have ambitions, ambitions to make metric capture and monitoring a more integral part of test runs.

I want test runners to capture those metrics. That's either by setting up the services to feed the metrics to the test runner itself, capturing the metrics directly by polling servlet interfaces, or capturing them indirectly via the cluster management tools.

Initially that'll just be a series of snapshots over time, but really, we could go beyond that and include in test reports the actual view of the metrics: what happened to various values over time? when when Yarn timeline server says its average CPU was at 85%, what was the spark history server saying its cache eviction rate was?

Now, what are the flaws in this idea?

There's the usual issue: the metrics we developers put in aren't what the operations team need. That's inevitable, but at least we are adding lots of metrics into the internal state of the system, and once you start instrumenting your code, you are more motivated to continue to add the others. It can't be worse than today.

The metrics published for monitoring tools aren't the ones we developers need. I've already hit this.

Problem 1: Binary data
I want to publish a binary metric: has the slider App Master had a node map update event from the YARN RM? That's a bool, not the usual long value metrics tools like.

The fix there is obvious for anyone who has programmed in C:

```java

public class BoolMetric extends AtomicBoolean implements Metric, Gauge<Integer> {

 @Override
 public Integer getValue() {
	return get() ? 1 : 0;
 }
```

It's not much use as a metric, except in that case that you are trying to look at system state and see what's wrong. It actually turns out that you don't get an initial map —something which GETS off the codahale JSON metric servlet did pick up in a minicluster test. It's already paid for itself. I'm happy. It's just it shows the mismatch between what is needed to monitor a running app, things you can have triggers and graphs of, and simple bool state view.

Problem 2: timestamps.

I want to track when an update happened, especially relative to other events across the system.

I don't see (in the codahale metrics) any explicit notion of time as a metric, other than histograms of performance. I want to publish a wall time, somehow. Which leaves me with two options. (a) A counter listing the time in milliseconds *when* something happened. (b) A counter listing the time in milliseconds *since* something happened. From a monitoring perspective, (b) is better: you could set an alarm if the counter value went over an hour. From a developer perspective, absolute values are easier to test with. They also support the value "never" better, with something "-1" being a good one here. I don't know what value of "never" would be good in a time-since-event value which couldn't be misinterpreted by monitoring tools. A value of -1 could be construed as good, though if it had been in that state for long enough, it becomes bad. Similarly, starting off with LONG_MAX as the value would set alarms off immediately. Oh, and either way, the time isn't displayed as a human readable string.

In this case I'm using absolute times. And thinking that if monitoring tools don't monitor time-since metrics and can be set to react when they exceed a value, that's something they should be doing. (Actually, I'm thinking of writing a timestamp class that publishes an absolute time on one path, and a relative time on an adjacent path. Something for everyone)

|The other issue is simply an execution one: wiring the metrics up, unwiring them at the end of test runs, naming them, etc. For unit tests, I don't register the metrics. For production, I do want names, and I want names constant over time. For the spark history server, there's only one cache, so easy. For Slider, we now have per-role metrics (instances, failures, etc): I'm just using the name of the role as the prefix here. For other bits of code: I'll just have to work it out as I go along.


I've been trying to find out who else has done this, and what worked/didn't worked, but there doesn't seem too much in published work. There's a lot of coverage of performance testing —but this isn't that. This about a philosophy of instrumenting code for unit and system tests, using metrics as that instrumentation —and in doing so not only enabling better assertions to be made about the state of the System Under Test, but hopefully providing more information for monitoring and diagnostics.

## Feedback of this proposal

Having sought feedback on this topic, especially from of the former LinkedIn team, some key points were raised.

A repeatedly highlit risk was that of exposing internal state —and in doing so, potentially creating metrics which the operations team end up depending on. Developers are then at risk of either unintentionally breaking metrics when they change implementation details, or of being constrained in changes. Essentially: metrics have to be treated as part of the public API of a service —an API— and designed and maintained accordingly.

Another risk is metric overload: publishing too many metrics of no value to operations
is actually harmful, as it adds extra strain on the system, as well as distracting those operations teams with the effort of deciding which metrics are in fact relevant and useful.

Finally, there's a performance overhead of using atomic and even `volatile` datatypes in Java code: you don't want to be synchronizing everything, or forcing memory barriers into the generated native machine code.

All of these push back against the idea of "exposing internal state as metrics", and instead advocate one of "have a well designed public view of state, one which is visible as a set of metrics". Tests querying the state of the system by way of the metrics are then, simply, making an API call of a different aspect of the service: it's metrics API.

Regarding the performance hit, note that as any `synchronized` clause is a memory barrier function where memory accesses will not be re-ordered above and below, you *should* be able to use non-volatile, non-atomic values where the application code itself can do this —and make the metric getter call the one which keeps the data up to date simply by tagging as `synchronized`. 




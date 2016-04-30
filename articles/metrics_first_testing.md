# Metrics First testing

## April, 2016


## Summary

In this article I introduce the concept of Metrics First Testing, and show how instrumenting the internals of classes, enabling them to be published as metrics, enables better testing of distributed systems, while also offering potential to provide more information in production.

Exporting instrumented classes in the form of remotely accessible metrics permits test runners to query the state of the **System Under Test**, both to make assertions about its state, and to collect histories and snapshots of its state for post-run diagnostics.

This same observable state *may be useful* in production —though there is currently no evidence to support this hypothesis.

There are a number of issues with the concept. A key one is if these metrics do provide useful in production, then they become part of the public API of the system, and must be supported across future versions.

## Introduction: Metrics-first testing 

Recently I've been doing more [scalatest](http://www.scalatest.org/) work, as part of SPARK-7889, SPARK-1537, SPARK-7481. Alongside that, in SLIDER-82, anti-affine work placement across a YARN cluster, And, most recently, wrapping up S3a performance and robustness for Hadoop 2.8, [HADOOP-11694](https://issues.apache.org/jira/browse/HADOOP-11694), where the cost of an HTTP reconnect appears on a par with reading 800KB of data, meaning: you are better off reading ahead than breaking a connection on any forward seek under ~900KB. (that's transatlantic to an 80MB FTTC connection; setup time is fixed, TCP slow start also means that the longer the connection is held, the better the bandwidth gets)

On these projects, I've been exploring the notion of *metrics-first testing*. That is: your code uses metric counters as a way of exposing the observable state of the core classes, and then tests can query those metrics, either at the API level or via web views.

Here's a test for [HADOOP-13047](https://issues.apache.org/jira/browse/HADOOP-13047): *S3a Forward seek in stream length to be configurable*

    @Test
     public void testReadAheadDefault() throws Throwable {
       describe("Verify that a series of forward skips within the readahead" +
           " range do not close and reopen the stream");
       executeSeekReadSequence(32768, 65536);
       assertEquals("open operations in " + streamStatistics,
           1, streamStatistics.openOperations);
     }

Here's the output

    testReadAheadDefault: Verify that a series of forward skips within the readahead
     range do not close and reopen the stream

    2016-04-26 11:54:25,549  Reading 623 blocks, readahead = 65536
    2016-04-26 11:54:29,968  Duration of Time to execute 623 seeks of distance 32768
    with readahead = 65536: 4,418,524,000 nS
    2016-04-26 11:54:29,968  Time per IOP: 7,092,333 nS
    2016-04-26 11:54:29,969  Effective bandwidth 0.000141 MB/S
    2016-04-26 11:54:29,970  StreamStatistics{OpenOperations=1, CloseOperations=0,
     Closed=0, Aborted=0, SeekOperations=622, ReadExceptions=0, ForwardSeekOperations=622,
     BackwardSeekOperations=0, BytesSkippedOnSeek=20381074, BytesRead=20381697,
     BytesRead excluding skipped=623, ReadOperations=0, ReadsIncomplete=0}


I'm collecting internal metrics of a stream, and using that to make assertions about the correctness of the code. Here, that if I set the readahead range to 64K, then a series of seek and read operations stream through the file, rather than break and reconnect the HTTPS link.
This matters a lot, as shown by one of the other tests, which times an `open()` call as well as that to actually read the data

    testTimeToOpenAndReadWholeFileByByte: Open the test file
     s3a://landsat-pds/scene_list.gz and read it byte by byte

    2016-04-26 11:54:47,518 Duration of Open stream: 181,732,000 nS
    2016-04-26 11:54:51,688 Duration of Time to read 20430493 bytes: 4,169,079,000 nS
    2016-04-26 11:54:51,688 Bandwidth = 4.900481  MB/S
    2016-04-26 11:54:51,688 An open() call has the equivalent duration of
     reading 890,843 bytes

Now here's a Spark test using the same source file and s3a connector

     ctest("CSVgz", "Read compressed CSV", "") {
       val source = sceneList
       sc = new SparkContext("local", "test", newSparkConf(source))
       val sceneInfo = getFS(source).getFileStatus(source)
       logInfo(s"Compressed size = ${sceneInfo.getLen}")
       val input = sc.textFile(source.toString)
       val (count, started, time) = duration2 {
         input.count()
       }
       logInfo(s" size of $source = $count rows read in $time nS")
       assert(ExpectedSceneListLines &lt;= count)
       logInfo(s"Filesystem statistics ${getFS(source)}")
     }


Which produces, along with the noise of a local spark run, some details:

    2016-04-26 12:08:25,901  executor.Executor Running task 0.0 in stage 0.0 (TID 0)
    2016-04-26 12:08:25,924  rdd.HadoopRDD Input split: s3a://landsat-pds/scene_list.gz:0+20430493
    2016-04-26 12:08:26,107  compress.CodecPool - Got brand-new decompressor [.gz]
    2016-04-26 12:08:32,304  executor.Executor Finished task 0.0 in stage 0.0 (TID 0). 
     2643 bytes result sent to driver
    2016-04-26 12:08:32,311  scheduler.TaskSetManager Finished task 0.0 in stage 0.0 (TID 0)
     in 6434 ms on localhost (1/1)
    2016-04-26 12:08:32,312  scheduler.TaskSchedulerImpl Removed TaskSet 0.0, whose tasks
     have all completed, from pool 
    2016-04-26 12:08:32,315  scheduler.DAGScheduler ResultStage 0 finished in 6.447 s
    2016-04-26 12:08:32,319  scheduler.DAGScheduler Job 0 finished took 6.560166 s
    2016-04-26 12:08:32,320  s3.S3aIOSuite  size of s3a://landsat-pds/scene_list.gz = 464105
     rows read in 6779125000 nS

    2016-04-26 12:08:32,324 s3.S3aIOSuite Filesystem statistics
     S3AFileSystem{uri=s3a://landsat-pds,
     workingDir=s3a://landsat-pds/user/stevel,
     partSize=104857600, enableMultiObjectsDelete=true,
     multiPartThreshold=2147483647,
     statistics {
       20430493 bytes read,
        0 bytes written,
        3 read ops,
        0 large read ops,
        0 write ops},
        metrics {{Context=S3AFileSystem}
         {FileSystemId=29890500-aed6-4eb8-bb47-0c896a66aac2-landsat-pds}
         {fsURI=s3a://landsat-pds/scene_list.gz}
         {streamOpened=1}
         {streamCloseOperations=1}
         {streamClosed=1}
         {streamAborted=0}
         {streamSeekOperations=0}
         {streamReadExceptions=0}
         {streamForwardSeekOperations=0}
         {streamBackwardSeekOperations=0}
         {streamBytesSkippedOnSeek=0}
         {streamBytesRead=20430493}
         {streamReadOperations=1488}
         {streamReadFullyOperations=0}
         {streamReadOperationsIncomplete=1488}
         {files_created=0}
         {files_copied=0}
         {files_copied_bytes=0}
         {files_deleted=0}
         {directories_created=0}
         {directories_deleted=0}
         {ignored_errors=0} 
         }}


**What's going on here?**

I've instrumented `S3AInputStream`, instrumentation which is then returned to its `S3AFileSystem` instance. This
instrumentation can not only be logged, *it can be used in assertions*. As the FS statistics are actually Metrics2 data, they can
be collected from running applications.


By making the observable state of object instances real metric values, I can extend their observability from unit tests to system tests —all the way to live clusters.


1. This makes assertions on the state of remote services a simple matter of `GET /service/metrics/$metric` + parsing.
1. It ensures that the internal state of the system is visible for diagnostics of both test failures and production system problems. Here: how is the file being
 accessed? Is the spark code seeking too much —especially backwards? Were there any transient IO problems which were recovered from?
 These are things which the ops team may be grateful for in the future, as now there's more information about what is going on.
1. It encourages developers such as myself to write those metrics early, at the unit test time, because we can get immediate tangible benefit from their presence. We don't need to wait until there's some production-side crisis and then rush to hack in some more logging. Classes are instrumented from the outset. Indeed, in [SPARK-11373](https://issues.apache.org/jira/browse/SPARK-11373) I'm actually implementing the metrics publishing in the Spark History server —something the [SPARK-7889] (https://issues.apache.org/jira/browse/SPARK-7889) code is ready for.

Metrics-first testing, then, is instrumenting the code and publishing it for assertions in unit tests, and for downstream test suites.

I'm just starting to experiment with this metrics-first testing.

I have ambitions to make metric capture and monitoring a more integral part of test runs. In particular, I want test runners to capture those metrics. That's either by setting up the services to feed the metrics to the test runner itself, capturing the metrics directly by polling servlet interfaces, or capturing them indirectly via the cluster management tools.  Initially that'll just be a series of snapshots over time, but really, we could go beyond that and include in test reports the actual view of the metrics: what happened to various values over time? when when Yarn timeline server says its average CPU was at 85%, what was the spark history server saying its cache eviction rate was?  Similarly, those s3a counters are just initially for microbenchmarks under `hadoop-tools/hadoop-aws`, but they could be extended up the stack, through Hive and spark queries, to full applications. It'll be noisy, but hey, we've got tooling to deal with lots of machine parseable noise, as I call it: Apache Zeppelin. 


## What are the flaws in this concept

### Relevance of metrics beyond tests

There's the usual issue: the metrics we developers put in aren't what the operations team need. That's inevitable, but at least we are adding lots of metrics into the internal state of the system, and once you start instrumenting your code, you are more motivated to continue to add the others.  

### Representing Boolean values

I want to publish a binary metric: has the slider App Master had a node map update event from the YARN RM? That's a bool, not the usual long value metrics tools like.  The fix there is obvious for anyone who has programmed in C: 

    public class BoolMetric extends AtomicBoolean implements Metric, Gauge&lt;integer&gt; {

     @Override
     public Integer getValue() {
       return get() ? 1 : 0;
     }

It's not much use as a metric, except in that case that you are trying to look at system state and see what's wrong. It actually turns out that you don't get an initial map —something which GETs off the Coda Hale JSON metric servlet did pick up in a minicluster test. It's already paid for itself. I'm happy. It's just it shows the mismatch between what is needed to monitor a running app, things you can have triggers and graphs of, and simple bool state view. 

### Representing time

I want to track when an update happened, especially relative to other events across the system.  I don't see (in the Coda Hale metrics) any explicit notion of time other than histograms of performance. I want to publish a wall time, somehow. Which leaves me with two options. (a) A counter listing the time in milliseconds *when* something happened. (b) A counter listing the time in milliseconds *since* something happened.

From a monitoring perspective, (b) is better: you could set an alarm if the counter value went over an hour. From a developer perspective, absolute values are easier to test with. They also support the value "never" better, with something "-1" being a good one here. I don't know what value of "never" would be good in a time-since-event value which couldn't be misinterpreted by monitoring tools. A value of -1 could be construed as good, though if it had been in that state for long enough, it becomes bad. Similarly, starting off with LONG_MAX as the value would set alarms off immediately. Oh, and either way, the time isn't displayed as a human readable string.  In this case I'm using absolute times. And thinking that if monitoring tools don't monitor time-since metrics and can be set to react when they exceed a value, that's something they should be doing. 

I'm thinking of writing a timestamp class that publishes an absolute time on one path, and a relative time on an adjacent path. Something for everyone. 

The other issue is simply an execution one: wiring the metrics up, unwiring them at the end of test runs, naming them, etc. For unit tests, I don't register the metrics. For production, I do want names, and I want names constant over time. For the spark history server, there's only one cache, so easy. For Slider, we now have per-role metrics (instances, failures, etc): I'm just using the name of the role as the prefix here. For other bits of code: I'll just have to work it out as I go along.  


### The performance of `AtomicLong` types

Java volatile variables are slightly more expensive than C++ ones, as they act as barrier operations rather than just telling the compiler never to cache them. But they are still simple types.  In contrast, Atomic* are big bits of Java code, with lots of contention if many threads try to update some metric. This is why Coda Hale use a an AtomicAccumulator class, one that [eventually surfaces in Java 8.](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/atomic/AtomicInteger.html). But while having reduced contention, that's still a piece of java code trying to acquire and release locks.  *It would only take a small change in the JRE for `volatile`, or perhaps some variant, `atomic` to implement atomic ++ and += calls at the machine code level, so the cost of incrementing a volatile would be almost the same as setting it*.  We have to assume that Sun didn't do that in 1995-6 as they were targeting 8 bit machines, where even incrementing a 16 bit `short` value was not something all CPUs could guarantee to do atomically. Now? Even watches come with 32 bit CPUs, phones are 64 bit. It's time for Oracle to look ahead and conclude that it's time for even 64 bit volatile addition to made atomic.  For now, I'm making some of the counters which I know are only being updated within thread-safe code (or code that says "should only be used in one thread") volatile; querying them won't hold up the system.   


### Pressure to align your counters into a bigger piece of work

For the S3a code, this surfaces in [HDFS-10175](https://issues.apache.org/jira/browse/HDFS-10175); a proposal to make more of those FS level stats visible, so that at the end of an SQL query run, you can get aggregate stats on what all filesystems have been up to. I do think this is admirable, and with the costs of an S3 HTTP reconnect being 0.1s, it's good to know how many there are. But at the same time, these overreaching goals shouldn't be an excuse to hold up the low-level counters and optimisations which can be done at a micro level —what they do say is "don't make this per-class stuff public" until we can do it consistently. The challenge then becomes technical: how to collect metrics which would segue into the bigger piece of work, are useful on their own, and which don't create a long term commitment of API maintenance.

### Over-instrumentation

As described by Jakob Homan: 

>Large distributed systems can overwhelm metrics aggregators; For instance, Samza jobs generated so many metrics LI's internal system blanched and we had to add a feature to blacklist whole metric classes 

These low-level metrics may be utterly irrelevant to most processes, yet, if published and recorded, will add extra load to the monitoring infrastructure. Again, this argues for making the low-level metrics off by default, unless explicitly enabled by a debugging switch.  In fact, it almost argues for having some metric enabling profile similar to log4J settings, where you could turn on, say, the S3a metrics at DEBUG level for a run, leaving it off elsewhere. That could be something to investigate further. Perhaps I could start by actually using the log level of the classes as the cue to determine which metrics to register:  

    if (LOG.isDebugEnabled) {
       registerInternalMetrics();
    }


### Brittle to refactoring

This an issue which surfaced during maintenance, when breaking up a class into three separate classes, each with the relevant set of metrics pulled out of the original.

This refactoring broke those tests which were directly referencing the variables. Although the tests could be updated, or the original class extended with delegating accessors, both actions incur costs *and remain brittle against future change*

I'll cover my final solution in detail, but essentially: making access to metrics via map lookup operations, passing in the string value of the metric. Delegation becomes a matter of extending the main class's map with those of the associated instances of the now-refactored classes. As well as making the tests less brittle, it does enforce keeping the names of metrics invariant, which is good for external consumption.

### Metrics are part of your public API

This is the troublesome one: If you start exporting information which your ops team depends on, then you can't remove it. (Wittenauer, a reviewer of a draft of this article, made that point quite clearly).  And of course, you can't really tell which metrics end up being popular. Not unless you add metrics for that, and, well, you are down a slippery slope of meta-metrics at that point.  The real issue here becomes not exposing more information about the System Under Test, but exposing internal state which may change radically across versions.  What I'm initially thinking of doing here is having a switch to enable/disable registration of some of the more deeply internal state variables. The internal state of the components are not automatically visible in production, but can be turned on with a switch. That should at least make clear that some state is private. However, it may turn out that the metrics end up being invaluable during troubleshooting; something you may not discover until you take them away. Keeping an eye on troubleshooting runbooks and being involved in support calls will keep you honest there.  

## Related work

I've been trying to find out who else has done this, and what worked/didn't worked, but there doesn't seem too much in published work. There's a lot of coverage of performance testing —but this isn't that. This about a philosophy of instrumenting code for unit and system tests, using metrics as that instrumentation —and in doing so not only enabling better assertions to be made about the state of the System Under Test, but hopefully providing more information for production monitoring and diagnostics. 

Similarly, searches for "Metrics and Testing" tend to focus on Metrics of Test Processes: that is instrumenting the test process itself, and generating statistics. While useful, that doesn't directly add extra instrumentation to the production code.

In *Making Web Services That work* ([Loughran02]), I argued that were metrics tangibly useful to developers,

## Conclusions

In Distributed Testing, knowing more about state of the System Under Test aids both assertions and diagnostics. By instrumenting the code better, or simply making the existing state accessible as metrics, it becomes possible to query that state during test runs. This same instrumentation may then be useful in the System In Production —though that it is something which I currently lack data about.   

##  Acknowledgements

Thank you to the reviewers: Jakob Homan, Chris Douglas, Jay Kreps, Allen Wittenauer, Martin Kleppman. I don't think I've addressed all their feedback, especially Chris's (security, scope,+ others), and Jay went into detail on how structured logging would be superior —something I'll let him expound on in the Confluent blogs. Interestingly, I am exposing the s3a metrics as log data, —it lets me keep those metrics internal, and lets me see their values in Spark tests without changing that code. 
AW pointed out that I was clearly pretty naive in terms of what modern monitoring tools could do, and should do more research there: 

> On first blush, this really feels naïve as to the state of the art of  monitoring tools, especially in the commercial space where a lot of  machine learning is starting to take shape (e.g., Netuitive, Circonus,  probably Rocana, etc, etc).

Clearly I have to do this...


### Bibliography

[Loughran02] Loughran,S, *[Making Web Services that Work](http://www.hpl.hp.com/techreports/2002/HPL-2002-274.html)*, HP Laboratories, 2002.
 
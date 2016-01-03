# Why is so much of my life wasted waiting for test runs to complete?

I've spent the weekend enduring the pain of kerberos-related functional test failures, test runs that take time to finish, especially as its token expiry between deployed services which is the Source of Madness (copyright (c) 1988 MIT).

Anyone who follows me on Strava can infer when those runs take place as if its a long one, I've nipped down to the road bike on the turbo trainer and done a bit of exercise while waiting for the results.

Which is all well and good except for one point: _why do I have to wait so long?_

While a test is running, the different services in the cluster are all generating timestamped events, "log messages" as they usually known,  The code the test runner itself is also generating a stream of events, from any client-side code and wrapping JUnit/xUnit runners, again, tuples of (timestamp, thread, module, level, text) + implicitly (process, host). And of course there's the actual outcome of each test.

Why do I have to wait until the entire test run is completed for those results to appear?

There's no fundamental reason for that to be the case. It's just the way that the functional tests have evolved under the unit test runners, test runners designed to run short lived unit tests of little classes, runs where stdout and stderr were captured without any expectation of structured format. When `<junit>` completed individual test cases, it'd save the XML DOM build in memory to an XML file under build/tests. After Junit itself completed, the build.xml would have a `<junitreport>` task to map XML -> HTML in a wondrous piece of XSLT. 

Maven surefire does exactly the same thing, except it's build reporter doesn't make it easy to stream the results to both XML files and to the console at the same time.

The CI tooling: Cruise Control and its successors, of which Jenkins is the near-universal standard took those same XML reports and now generate their own statistics, and again wait for the reports to be generated at the end of the test run.

That means those of us who are waiting for a test to finish have a limited set of choices

1. Tweak the logging and output to the console, stare at it waiting to see stack traces to go by
2. Run a single failing test repeatedly until you fix it, again, staring at the output. In doing so you neglect the rest of the code until at the end of the day you are left with the choices of (a) run the hour long test of everything to make sure there are no regressions and (b) commit and push and expect a remote Jenkins to find the problem, at which point you may have broken a test and either need to get those results again & fix them, or rely on the goodwill of a colleage (special callout, Ted Yu, the person who usually ends up fixing SLIDER-1 issues)

Usually I drift into the single-test mode, but first you need to identify the problem. And even then, if the test takes a few minutes, each iteration hurts. And there's the hassle of collecting the logs, correlating events across machines and services to try and understand what's going on. If you want more detail, its over to http:{service host:port}/logLevel and tuning up the logs to capture more events on the next iteration, and so you are off again.

A hard-to-identify problem becomes a "very slow to identify problem", or productivity killer.

Sitting waiting for tests is a waste of time for software engineers.

What to do?

There's parallelisation. That would be great, but you are still waiting for the end of the test runs for the results, unless you are going to ssh into the hosts and play tail -f against log files, or grep for specific event texts.

What would be just as critical is: real time reporting of test results.

I've discussed before how we need to radically improve tests and test runners.

What we also have to recognise is that the test reporting infrastructure equally dates from the era of unit tests taking milliseconds, full test suites and XSL transformations of the results taking 10s of seconds at most.

The world has moved on from unit tests.

What do I want now? As well as the streaming out of those events in structured form directly to the some aggregrator, I want that test runner to be immediately publishing the aggregate event stream and test results to some viewer better than four consoles with tail -f streaming text files (or worse, XML reports). I want HTML pages as they come in, with my test report initially showing all tests enumerated, then filling up as tests run and fail. I'd like the runner to known (history, user input?) which tests were failing, and so run them first. If I checked in a patch to a specific test class, that'll be the one I want to run next, followed by everything else in the same module (assuming locality of test coverage).

Once I've got this, the CI tooling I'm going to run will change. It won't be a central machine or pool of them, it'll be a VM hosted locally or some cloud infrastructure. Probably the latter, so it won't be fighting for RAM and CPU time with the IDE.

Whenever I commit and push a patch to my private branch, the tests should run.

It's my own personal CI instance, it gets to run my tests, and I get to have a browser window open keeping track of the status while I get on with doing other things.

We can do this: its just the test runner reporting being switched from batch to streaming, with the HTML views adapting.

If we're building the largest distributed computing systems on the planet, we can't say that this is beyond us.
<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

# Roadkill


## Goals

Extend existing test running framework for functional testing, 


1. Support setup and teardowns as ordered sequence of closures
1. Support failure closures which are invoked on test failure:
grab reports of web pages &c.
1. Support scheduled closures which are invoked on a regular
basis through the thread to capture information like metrics.
1. Launch conditions: await observed state with backoff and fast-fail options.
1. Support configurability beyond maven properties
1. Kerberos/Secure clusters.
1. Output data to flume via avro IPC
1. Option to output data as avro records written to a file, then
 copied into a destination dir (with timestamp) for avro pickup.
1. Avro records for test run info as well as log events
1. "something" to take flume stream to build up test run
  (really the session is the test case) and then generate JUnit XML.
1. Output intermediate data to console.
1. Some spark streaming app to collect live data and report; something
to run over final results for better postmortems.

## What is a test?

* Something which gets the SUT into an observable state and makes
assertions about that state.
* Something which makes direct/indirect changes to the SUT and 
assert that observations of the SUT match expectations.
* Something which triggers failure modes of the underlying network/compute
infrastructure and, from observations of the SUT, that it's handling
of the events match requirements.


## What language/format for a test?

* The defacto standard for Java test runs is JUnit; for Scala scalatest,
along with various extensions for mocking (mockito, ...) and declarative
assertions (hamcrest, spock). Selenium is the standard mechanism for
testing HTTP behaviour. 

* Non-Java, non-scala System tests often use python and pyunit

* The Jepsen fault-inducing framework is in Clojure; that language
is also (I believe) used to specify tests.


*what is there for observing the state of emulated mobile applications
inside a test run*

Trying to define a new language/framework for test running would
be a distraction that cold 

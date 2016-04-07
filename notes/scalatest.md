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

# Scalatest: a review


## Nice features


### the `eventually()` construct

This takes a timeout, an interval (which can include policies like exponential retry),
and a closure to execute.

```scala
eventually(stdTimeout, stdInterval) {
  val entities = listEntities(queryClient)
  val entityNames = entities.map(_.getEntityId).mkString(" ")
  val maybeEntity1 = entities.find(_.getEntityId == attemptId1.toString)
  assertSome(maybeEntity1, s"Did not find attempt $attemptId1 in $entityNames")
  val entity = maybeEntity1.get
  val asAppHistory = toApplicationHistoryInfo(entity)
  assert (asAppHistory.completed,
    s"App never completed; history=$asAppHistory," +
    s"entity=${describeEntity(entity)}")
}
```

In practice, a couple of flaws have surfaced

1. There's no fail-fast option; if a probe detects that a condition will never be met, there's
no way to report this and have `eventually()` halt immediately. (actually, it is possible to define
a filter, once you look into the code). You could have specific assertions that are retryable,
other failures lead to immediate exit

1. All exceptions raised in the closure trigger a retry, rather than just an assertion
failure. If the test clause contains a bug, such as a null pointer dereference, class cast
failure or anything else, this failure will be retried until a timeout.

These could be addressed with a variant which (a) failed fast on anything other than an assertion
exception and (b) recognised a specific "fast-assert-failure" exception as a special case.


## Weaknesses


### Aborted tests aren't displayed as such in the xml/surefire reports

What's an aborted test? Setup failure. You know the test run failed, but when you look 
the surefire-generated report, it says 0-failures, 0-errors, 100% success.

### Short stack on assertion failures
 
You don't get much stack on an assertion failure. That may seem a feature for readability,
but if your assertion is inside some extended check method called from your test case,
you want the full test case details, not just the fact that it's in your new assertion
method.

### Limited output compared to groovy assert

Groovy gives a hierarchical toString dump of your assert, taking advantage of the fact
the raw script is there for interpretation.

Scala doesn't have that luxury.

The text used in assert failures, the "clue" argument, isn't a simple string, it can take a function
returning a string. This is how it avoids evaluating interpolated strings unless an assertion fails.
This means that it could be possible to have some fairly complex operations called on an assertion
failure, including potentially querying services for state, dumping the output, etc.



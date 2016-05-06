# Securing Hadoop Cloud Tests

Nobody wants to commit their AWS, Azure or Openstack credentials to an SCM repository, not when there are robots looking for your keys so they can spin up bitcoin mining VMs.


That's why [git-secrets](https://github.com/awslabs/git-secrets) is something you should consider preinstalling on all repos where you go near repositories. Unfortunately, it's pattern matching picks up a lot of invalid patterns in the Hadoop codebase, and a few in spark. In the absence of a decent pattern to only scan text files, after installing I edit out the regexps from .git/config and rely on it scanning purely for the strings in ~/.aws/credentials.


That keeps the keys out of SCM, if everyone is set up that way.


Which leaves the next problem: *if you save your hadoop configuration files to SCM, how do you get AWS keys into system configurations?*


The answer: [XInclude](https://www.w3.org/TR/xinclude/). This is one of those features of XML which got forgot about in the rush to embrace JSON: a mechanism to reference other XML documents for nested inclusion, documents which can be referenced by relative *or absolute* paths.


This is one of my configuration files, specifically `hadoop-tools/hadoop-aws/src/test/resources/auth-keys.xml`


This is a special file, tagged in Hadoop's `/.gitignore` to keep it out of the repos. Yet still the keys are in that source tree, still at risk of sneaking out.


To deal with this, stick an absolute XInclude reference into the file, pointing to the configuration file where the keys really live.

```xml
<configuration>
  <include xmlns="http://www.w3.org/2001/XInclude"
    href="file:///home/stevel/.aws/hadoop-keys.xml" />
</configuration>
```

This tells the Hadoop config loader to grab the keys from a file in `~/.aws`; one which lives carefully out of the SCM-managed space.


Provided the contents of that directory are kept private, my keys will not get checked in.

They can, however leak, in various ways, including

1. In the site configuration served up  `/config` URL of a service
1. Code which accidentally logs it.
1. If it gets saved to files/datasets used in bug reports.
1. Malicious code running in your system which grabs the keys and exports/logs them. This is why no OSS jenkins servers are set up with the keys needed to test against object store.

Clearly, more should be done to lock it down.


There are some patches under the S3A phase 2 JIRA to support credentials better, [HADOOP-12723](https://issues.apache.org/jira/browse/HADOOP-12723), [HADOOP-12537](https://issues.apache.org/jira/browse/HADOOP-12537). This is somewhere where anyone who can test these patches is invaluable. I'm currently doing the review-then-commit in S3, but I'm only doing it at weekends, when I have some spare time, and even then, as a full test run takes 2+ hours, not reviewing very much.

Anyone can review Hadoop patches, confirm whether they worked or not, show how they didn't. It's the way to verify that forthcoming code from other people works for you, and of contributing time and effort into the community. Other people have done the coding —time to help with the testing.

Especially now I've just documented how to keep the keys safe when you set up S3 for a test run. Once the auth-keys.xml file is in the `src/test/resources` directory, Maven tests s3, depending on the specific properties, s3a, s3n and s3a are all tested.


**Footnote #1 ** The path `~/.aws/hadoop-keys.xml` isn't where my credentials live, before someone thinks of an XInclude-based attack which somehow gets at them.

**Footnote #1 ** S3A now also supports KMS, which can also be referenced. These are a lot harder to accidentally get at in code, intentionally and unintentionally.


### History

2016-05-06 First published.
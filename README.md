# DeFlaker
More information at [http://www.deflaker.org](http://www.deflaker.org).

## Overview
Everything is maven. From the root directory, you can do `mvn install` to build and install, or `mvn eclipse:eclipse` to generate Eclipse .project files (so you can import those existing projects into eclipse and modify). There are no tests. Sorry. I have some git repositories outside of this one on my computer that I used for testing. Setting up that infrastructure properly is possible, and an exercise I leave to the reader.

## Installation and configuration
All that you *really* need is the extension, which is in maven central. Or, to build yourself, from this directory, run `mvn install`. There are two ways to configure a project to use the system:

### Installing for an individual project
Add the following to the project's pom.xml file:

```
<project>
...
<build>
...
		<extensions>
			<extension>
				<groupId>org.deflaker</groupId>
				<artifactId>deflaker-maven-extension</artifactId>
				<version>1.4</version>
			</extension>
		</extensions>
...
</build>
</project>
```

### Installing site-wide
Transforming POM files is stupid and annoying and the whole point of the extension is that it will figure out how to change all of the plugin and dependency configuration for you. So, you can just copy the extension jar into your maven installation's lib/ext folder, and it will dynamically insert itself on *every* build that you make (until you delete it from that folder). You can figure out where that folder is by doing 

```
jon@lrrr:~/Documents/GMU/Projects/surefire-diff-coverage$ mvn --version
Apache Maven 3.2.5 (12a6b3acb947671f09b81f49094c53f426d8cea1; 2014-12-14T12:29:23-05:00)
Maven home: /usr/local/Cellar/maven32/3.2.5/libexec
Java version: 1.8.0_101, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_101.jdk/Contents/Home/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "mac os x", version: "10.12.3", arch: "x86_64", family: "mac"
```

In that case, the directory would be `/usr/local/Cellar/maven32/3.2.5/libexec/lib/ext/`

### Configuration options and defaults
By default, DeFlaker  assumes that there is a .git directory in the same directory as the root POM file of the build being run. If that's false, you can override it by providing the absolute path to it as the `gitDir` system property when invoking maven (`mvn -DgitDir=...`). 

By default, DeFlaker computes the diff between the repository HEAD and its parent. This makes it easy to do an evaluation where you check out version X of a project then just run it on it. For actual developer usage, it probably makes more sense to diff the working-copy vs the HEAD. That should also be pretty easy as an exercise to the reader... You CAN specify a different git commit to compute the diff off of, but other than for testing, that seems like a Bad Thing (because you'll use the diff of version X with the tests of version Y)... but anyway, `-Dcommit=SomeCommit` should work for that purpose.

There are no other configuration options currently exposed. DeFlaker assumes that test results go into the `${basedir}/target/surefire-reports` directory. That could be extracted to an option if it's a problem.

By default, DeFlaker only cares about tests that fail. If you know that a test is flaky, and want to have it ALSO see if there were no diffs impacting that test, do `-Ddeflaker.knownFlakes=testClass1,testClass2,...`.


## Getting output
Now that you have deflaker installed, you'll collect coverage every time you run `mvn test` or `mvn verify` (it hooks into both surefire and failsafe). It will generate a (plain text) file in `$[basedir}/target/diffcov.log` that contains the following columns:

```
testClass	coveredClass	isCoverageClassLevel	lineNumber
```

There's also a binary file produced in `${basedir}/.diffCache` that is deleted at the start of each run which contains the git-diff information (cached for multi-module projects and for use by both the test agent and the coverage reporter).

The extension will *always* override your test configuration so that *tests can never break the build* and instead, will fail in the reporting phase if there was a broken test. The reporting only runs for `mvn verify` right now though, so, good to keep in mind...

When you run `mvn verify`, you'll even get some pretty output on your build log from the maven plugin examining your test results, which might look like this:

```
[INFO] ------------------------------------------------------------------------
[INFO] TEST DIFFCOV ANALYSIS
[INFO] ------------------------------------------------------------------------
[INFO] Using covFile: /Users/jon/Documents/GMU/Projects/surefire-diff-coverage/experiments/tested-project/target/diffcov.log
[INFO] Using difFile: /Users/jon/Documents/GMU/Projects/surefire-diff-coverage/experiments/tested-project/.diffCache
[WARNING] FLAKY>> Diff has gone untested: edu.gmu.swe.diffcov.test.App was not covered on changed line(s) [11, 24]
[ERROR] FLAKY>> Test edu.gmu.swe.diffcov.test.CopyOfAppTest failed, perhaps due to relevant changed code:
	edu.gmu.swe.diffcov.test.App:16
	edu.gmu.swe.diffcov.test.App:17
	edu.gmu.swe.diffcov.test.App:23

[ERROR] 
There are test failures!
```

or

```
[INFO] ------------------------------------------------------------------------
[INFO] TEST DIFFCOV ANALYSIS
[INFO] ------------------------------------------------------------------------
[INFO] Using covFile: /Users/jon/Documents/GMU/Projects/surefire-diff-coverage/experiments/tested-project/target/diffcov.log
[INFO] Using difFile: /Users/jon/Documents/GMU/Projects/surefire-diff-coverage/experiments/tested-project/.diffCache
[WARNING] QUALITY>> Diff has gone untested: edu.gmu.swe.diffcov.test.App was not covered on changed line(s) [11, 16, 17, 24]
[WARNING] FLAKY>> Test edu.gmu.swe.diffcov.test.CopyOfAppTest failed, but did not appear to run any changed code
[ERROR] 
There are test failures!
```

# Nasty bits and limitations
For multi-module projects, you'll get 1 execution of the DeFlaker reporter *per module with tests* but it will still consider code in other modules (not that smart)... so probably best to ignore the "uncovered" warnings in those cases (this fix should be easy, and again, I leave to the astute reader). For the "flaky" designation, this shouldn't be a problem though. In general, the focus of this was of course on setting it up to detect flaky tests, not on building an awesome coverage tool (perhaps I leave that as an exercise to myself for a deadline further out..).

# Performance measurements
The overhead should be negligible, other than the fact that we have to run a `git diff` basically once before running tests - roughly a fixed cost (so if you only have 0.5msec of tests, yes, it will be much slower). To easily collect timing data, you can use [another maven extension of mine](https://github.com/jon-bell/maven-lifecycle-logger), that will collect msec level timing of each goal/phase/mojo execution. You should see the majority of the penalty occurring in the first execution of surefire/failsafe in the project (when diffs are computed).

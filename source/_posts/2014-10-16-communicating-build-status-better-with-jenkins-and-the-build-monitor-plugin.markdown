---
layout: post
title: "Communicating build status better with Jenkins and the Build Monitor plugin"
author: Jan Molak
date: 2014-10-16 17:21:49 +0100
published: true
comments: true
categories:
- continuous integration
- build monitor
- jenkins
---

If you're doing continuous integration chances are that you also care
about your team understanding the current state of your project.
Maybe you use a lava lamp or some other fancy device to visualise whether your
builds are red or green.
Or perhaps you've got my little [Build Monitor plugin](https://github.com/jan-molak/jenkins-build-monitor-plugin)
installed on your Jenkins box and displayed on some shared screen for your team to see?

Most visual extreme feedback devices use similar representation of the build status
- the well-known [RAG rating](http://en.wikipedia.org/wiki/Traffic_light_rating_system) used in a traffic light system.
Now, what does this RAG rating mean and how will it be interpreted by your team?
Yes, I know that you understand how the traffic lights work, but bear with me for just a moment
and maybe this short article will help you to avoid the bad case of rotting builds.

<!-- more -->

**RAG** stands for **R**ed, **A**mber and **G**reen.

**Green** is simple - it indicates that everything is OK:
your tests have passed,
deployment succeeded,
an artifact has been correctly produced and it matches your quality criteria,
this sort of thing. Basically it's safe for the traffic to proceed and for you to introduce new awesome features.

**Red** is also pretty straight-forward - one of the above is not the case. The traffic should stop.
Perhaps your tests have failed? Or maybe your CI server couldn't promote the artifact
for some reason? One way or another, red means that something didn't go according to plan
and someone should look into it _immediately_.

**Amber** - now that's the one I have a problem with...

## The bad case of rotting builds

In Jenkins, amber represents an _unstable_ build.
OK, but first, what does _unstable_ mean according to the [Jenkins terminology](https://wiki.jenkins-ci.org/display/JENKINS/Terminology)?

{% blockquote Jenkins Terminology, Table of terms used in Jenkins %}
A build is unstable if it was built successfully and one or more publishers report it unstable. For example if the JUnit publisher is configured and a test fails then the build will be marked unstable.
{% endblockquote %}

And here's the trouble, you see? To me if a test has failed, it means that either the software I&nbsp;help building
no longer behaves as expected or the test case no longer reflects what the desired behaviour is.
Both scenarios should be reflected in a broken, red build, and a red build should result in me stopping whatever new features
I was adding and fixing what's broken first.

Another trouble with using amber to represent builds with failing tests is that
humans are optimistic by nature[^1] - amber simply doesn't look as bad in the status report
as red so you can get away with it for longer.

I've seen development teams ignoring _unstable_ builds for weeks and even months because
"they were not _really_ broken",
"only had _a few_ flaky tests"
and
"[those failing tests] were not _that_ important anyway".
On several occasions I've worked with teams that started with accepting a small percentage of their tests failing as "normal and to be expected".
Over time, however, this small percentage became 10%, then 15% then 50%, eventually ignoring test failures got them to a stage where those poor chaps
started to question the point of automated testing altogether.

If you're using Jenkins to run automated tests I strongly encourage you to consider the [Broken Windows Theory](http://en.wikipedia.org/wiki/Broken_windows_theory),
fail the build when the tests fail and make sure you fix those failures as soon as you learn about them.

OK, so why have I [recently introduced](http://bit.ly/JBMBuild132) a visual representation of an _unstable_ build to the [Build Monitor plugin](https://github.com/jan-molak/jenkins-build-monitor-plugin)?
I've done it after a [long conversation](https://github.com/jan-molak/jenkins-build-monitor-plugin/issues/9) with the Build Monitor plugin community because...

## Not all Jenkins jobs are created equal

... and not all of them execute automated tests. Consider a Jenkins job gathering static code analysis metrics for example. Let's take test coverage for argument's sake.

For some development teams this metric falling below say 80% is an important problem that needs to be addressed immediately (a red build).
Other teams prefer to shoo before they shoot: use amber to indicate the metric falling below the required, ideal level
(_the coverage has decreased, we should look into it when we have a moment_)
and distinguish it from it falling below say 50% (_did someone ignore all the tests again?_)
so that they can prioritise their work better.

Remember however that even in the above scenario, if you leave your _unstable_ builds all by themselves for too long,
they will rot and cause you trouble when least expected[^2].

Happy green builds!

[^1]: to all the pessimist out there - here's the research done by the University of Kansas and Gallup ;) - [People By Nature Are Universally Optimistic, Study Shows](http://www.sciencedaily.com/releases/2009/05/090524122539.htm)
[^2]: I'm sure you know of the [Murphy's law](http://www.murphys-laws.com/murphy/murphy-true.html)
---
layout: post
title: "Restructuring Accumulo internals (Part 1)."
date: 2016-07-12 12:43:00
---

Motivation
----------

Having had the chance to develop a bit on Accumulo, I've come to the realization
that Accumulo suffers from a serious lack of solid internal abstractions in 
many places.  Additionally, some of the practices commonly used in Accumulo 
development make Accumulo more difficult to test, and more difficult to change.

I'm thinking about reorganizing, refactoring, or rewriting portions of Accumulo
to improve abstractions and reduce or eliminate harmful practices.  

Primary goals:

* Better testability, particular unit testability
* Better separation of concerns
* Looser coupling

Anti-goals:

* Performance is important, but not at the cost of an unmodifiable code base. 

Issues
------

Static singletons
=================
Within Accumulo, the [Singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern) 
abounds.  Singletons are an important piece of practically any complex object-oriented 
software.  Binding singletons to global variables is a simple and straightforward method 
for ensuring only one instance of a particular singleton exists.

Unfortunately, sharing state via global variables makes it impossible to isolate
units from these variables.  As such, it becomes very difficult to create unit
tests for most parts of Accumulo.  You either test the whole thing (an
integration test), or test nothing.  As a partial consequence of this, Accumulo
has a very slow and top-heavy test suite.

Lifecycle management
====================
While Accumulo offers tools to manage server process lifecycles, there are no
corresponding tools for Accumulo developers to call upon to manage Accumulo
object lifecycles.  Of particular note to me, Accumulo code creates threads and
thread pools in a very undisciplined manner.  There is no way to request that a
constructed Accumulo object graph release all of its non-memory resources (like
threads) in a timely manner in preparation for shutdown.

In general performing a "clean" shutdown of a complex application is a hard
problem, and one I'm not necessarily advocating we spend significant resources
on.  Indeed, I'm an advocate of [Crash-Only Software](https://www.usenix.org/legacy/events/hotos03/tech/full_papers/candea/candea.pdf)
design, whose central tenet can be stated as: the only way to shutdown
the application is to crash it; the only way to start the application is to
initiate crash recovery."

But there are two scenarios which I'd like to see Accumulo support:

* A user of the Accumulo client library should be able to instruct the library
  to release all threads (and other non-memory resources) in a timely manner.  
  Such a capability is essential for software run within a multitenant container
  (e.g. Tomcat, JBoss).
* Single JVM (partial) integration testing.  Currently most of Accumulo's
  integration tests use `MiniAccumuloCluster`, and depend on process
  isolation to ensure that the test framework can recover resources.  This
  choice makes it more difficult to debug tests.

Component Architecture
======================
The internal component architecture of Accumulo servers suffers from a number of
code smells and antipatterns:

* Tight coupling
* Poor separation of concerns
* [The Blob](https://sourcemaking.com/antipatterns/the-blob)/"God class" antipattern
  (notably `master.Master` and `tserver.TabletServer`).
* [Lava Flow](https://sourcemaking.com/antipatterns/lava-flow) antipattern /
  "Dead code" code smell. Another developer related to me the other day why 
  Accumulo sports two classes named `ZooReaderWriter` and two classes named
  `ZooCache`.  A story which boiled down to "The issue of removing one 
  of each pair didn't help address the problem being solved, and so was not attempted."
* [Shotgun Surgery](https://sourcemaking.com/refactoring/smells/shotgun-surgery)
  code smell.

Up next
-------
In [Part 2](/2016/07/12/Restructuring-Accumulo-Internals_part_2.html), I outline my
proposal for addressing these issues.

---
layout: post
title: "Restructuring Accumulo internals."
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
on.  Indeed, I'm an advocate of [Crash-Only Software](https://www.usenix.org/legacy/events/hotos03/tech/full_papers/candea/candea.pdf) design.

But there are two scenarios which I'd like to see Accumulo support:

* A user of the Accumulo client library should be able to instruct the library
  to release all threads (and other non-memory resources) in a timely manner.  
  Such a capability is essential for software run within a multitenant container
  (e.g. Tomcat, JBoss).
* Single JVM (partial) integration testing.  Currently most of Accumulo's
  integration tests depend upon `MiniAccumuloCluster`, and depends on process
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
  "Dead code" code smell.
  Another developer related to me the other day why Accumulo sports two classes
named `ZooReaderWriter` and two classes named `ZooCache`.  A story which boiled
down to "The issue of removing one of each pair didn't help address the problem being solved,
and so was not attempted."
* [Shotgun Surgery](https://sourcemaking.com/refactoring/smells/shotgun-surgery)
  code smell.


Proposal
--------

To address the above concerns, I'm proposing two related changes: Introducing
dependency injection, and reorganizing Accumulo into a partially tiered
architecture.

Dependency Injection
====================
The main thrust of my proposal is that Accumulo make use of [dependency
injection](https://en.wikipedia.org/wiki/Dependency_injection) to tie components
together.

In particular, I propose we use Google's
[Guice](https://github.com/google/guice/blob/master/README.md) framework to
manage the construction and wiring of Accumulo's object graph(s), and replace the
use of both manual dependency injection and static singletons.

Some investigation leads me to believe that the moving Accumulo to using
dependency injection won't be straightforward.  Largely due to poor separation
of concerns.  

For example, each of Accumulo's server objects runs
`Accumulo.init(...)` in its main routine.  `Accumulo.init(...)` runs around and
touches on a whole bunch of separate initialization concerns, whose proper place
is elsewhere.  One job of `Accumulo.init(...)` is to wait ZooKeeper to be ready; 
ZooKeeper access objects should take this responsibility. 

As such, I believe to make best use of dependency injection, we will need to
redesign portions of Accumulo to better separate concerns.

Partially Tiered Architecture
=============================
Better separation of concerns is a worthy goal on its own.  I don't have
anything approaching a complete design in mind, but rather a sketch of a design.

I propose arranging Accumulo's internals into a partially tiered architecture.
Objects within a tier may communicate freely with/depend upon objects in lower tiers, and only indirectly with objects in higher tiers.

Something roughly like this:

![Proposed Architecture
Sketch](/assets/AccumuloDesignSketch.png){:class="img-responsive"}

In this sketch, I've imagined 4 tiers.  From top to bottom:
* Exposed interfaces: Objects which coordinate the outside world's interaction
  with Accumulo or with an Accumulo server process.  These will often make
  changes directly to models, but may also spawn new actors (or invoke existing
  ones).
* Actor tier: Actors work with models and coordinate state-based changes
  affecting one or more model.  Actors may either poll models for changes, or
  receive notification via events or the (Observer pattern)[https://en.wikipedia.org/wiki/Observer_pattern).
* Model tier: Models are primarily concerned with maintaining state.  They
  expose interfaces to query and modify that state, and emit events or make
  callbacks to notify interested actors about state changes.
* Foundation tier: Fundamental software tools that are essentially independent
  of Accumulo.  Managed thread pools, logging, event delivery.

Challenges & Costs
------------------

Status Quo Antebellum
=====================

The Journey
===========

The Destination
===============


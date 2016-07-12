---
layout: post
title: "Restructuring Accumulo internals (Part 2)."
date: 2016-07-12 15:00:00
---

Recap
-----
In [Part 1](/2016/07/12/Restructuring-Accumulo-Internals_part_1.html), I touched on a group of issues I wanted to address with Accumulo:

* Static singletons
* Lifecycle management
* Component architecture.

Next, I want to touch on a proposal to address these issues.

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
manage the construction and wiring of Accumulo's object graph(s).

With the adoption of dependency injection I hope to address the issues of:

* Static singletons: Guice will be given the responsibility of maintaining
  references to singletons and distributing them to objects which need them.
  Guice can be configured to substitute some or all of those objects with mocks
  or stubs.
* Lifecycle management: With some conservative additions to Guice, and some
  significant restructuring of Accumulo, it should be possible to build a
  sufficiently powerful object lifecycle management framework.  Or we could just
  steal one, like Netflix's [Governator](https://github.com/Netflix/governator)
  library.
* Single JVM integration testing: A single instance of Guice's `Injector` class
  would be used to manage an Accumulo server process's entire object graph.  The
  `Injector` instances share no state.  So within a single JVM, one `Injector`
  instance could be configured to be a Tablet Server, and another `Injector`
  instance could be configured to be a Master, etc.  With the elimination of
  static singletons, and the introduction of sufficient lifecycle management, it
  should be possible to perform some types of tests currently requiring a
  MiniAccumuloCluster within a single JVM.

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

Up Next
-------
In part 3, I outline a roadmap for the changes in this proposal, and discuss some of the challenges and costs involved along the way.


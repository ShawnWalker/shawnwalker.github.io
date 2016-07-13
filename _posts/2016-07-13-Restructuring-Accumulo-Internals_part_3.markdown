---
layout: post
title: "Restructuring Accumulo internals (Part 3)."
date: 2016-07-13 12:00:00
---

Recap
-----
In [Part 1](/2016/07/12/Restructuring-Accumulo-Internals_part_1.html), I
touched on a group of issues I wanted to address with Accumulo:

* Static singletons
* Lifecycle management
* Component architecture.

In [Part 2](/2016/07/12/Restructuring-Accumulo-Internals_part_2.html), I proposed
a direction for addressing these issues, by restructuring Accumulo to use:

* Dependency injection.
* Tiered architecture. 

In this part, I sketch out a roadmap for this proposal, and discuss some of the
challenges and costs involved.

Roadmap
-------

1. Introduce dependency injection framework.  Build the pieces needed to use 
  Guice effectively.
 * Wiring code structure.  Guice dictates that the wiring of the object graph
   is largely directed by implementations of of the `Module` interface; examples
   [here](https://github.com/google/guice/wiki/GettingStarted).  As I'll mention
   in the challenges section, organizing these effectively is important for testability.
 * Lifecycle management skeleton.
2. Create *temporary* wiring for commonly used static singletons (e.g. `SiteConfiguration`)
   which arranges for the instances to be retrieved from the existing static factory methods.
3. Implement and wire new foundation layer components:
 * ThreadPools/ThreadPool factories
 * EventBus(es)
4. Iteratively move responsibility for creation of objects into the DI framework.  This
   proceeds roughly as a topological sort among components.  Find a component with 
   no dependees other than those already managed by DI, and move responsibility 
   for its creation into the DI framework.  Then go fix up the dependees to acquire 
   the component via dependency injection instead of via `new FooBarComponent(...)` or 
   `FooBarComponent.getInstance(...)`.
 * Generally speaking, classes which are just wrappers for data would remain outside
   the purvue of the DI framework.  For example, to get an instance of `KeyExtent`
   representing a specific tablet, one would write still write `new KeyExtent(...)`.
 * Logging will probably remain outside of the DI framework as well.  In my vision, 
   the logger's proper place is as a foundation-tier element of the object graph.  
   That said, the cost of moving logging into DI framework is significant, and 
   the potential gain from being able to replace the logger during testing is minimal.
5. As components become candidates for inclusion, refactor or rewrite those 
   components which don't fit into the partially tiered architecture.  Particularly 
   for those components which resist DI because of circular dependencies or 
   overly tight coupling.

Costs/Risks
-----------
1. Elimination of static singletons may break existing implementations
   of pluggable components such as iterators, `TabletBalancers`, or `VolumeChoosers` if
   they depend upon the ability to grab hold of parts of Accumulo's object graph via
   the corresponding static factory methods.
2. Manual dependency injection is verbose.  Tedious. Using a DI framework like Guice 
   avoids much of the verbosity, but exchanges it for complexity.  To a developer 
   not familiar with your project, Guice very much seems to magically make 
   objects appear out of thin air, with no hint of where they came from.
3. The iterative changes suggested by my proposal imply many distinct commits over a
   long timeframe.  During this process, Accumulo will be left with the worst of both
   worlds: most of the additional cost/complexity comes at the beginning with the
   introduction of DI, while the benefits accrue only over time.

   While this might suggest development in a separate isolated branch, I'm
   opposed to such an idea.  I'm proposing significant refactoring or even rewriting
   across much of Accumulo's code base over time; as such, performing a merge after
   6 months' or a year's worth of (wildly divergent) changes would likely be all but 
   impossible.

   Going further, merging a single change set against a component whose
   interface and implementation had significantly shifted might require work similar
   to completely reimplementing the change set.  As such, I believe an approach involving
   making periodic merges from the "master" branch into the "refactor" branch would 
   be infeasible as well.

Challenges
----------
1. Dependency injection is about decoupling individual classes from their dependencies, 
   which enables unit testing.  Components made of multiple cooperating objects can
   be more difficult to handle. Convincing Guice to construct the object graphs of 
   individual components requires careful modular design of the wiring modules, and is 
   not well supported in Guice's stock wiring EDSL.
2. Ideally, a component in an Object Oriented application should be ready to use
   as soon as it is completely constructed (and possibly initialized).  Unfortunately,
   many concerns in Accumulo are currently written in a procedural and strictly ordered 
   fashion, where a procedure has responsibility for executing a collection of actions 
   touching multiple components in a specific order. Some such concerns will be more 
   appropriately represented by distributing initialization and state change among 
   components.  Others will be more problematic.  Examples of such ordered concerns:
 * `Accumulo.init()` with various initialization concerns.
 * Master's upgrade process.
 * Master's and TServer's state automata.

3. `TabletServer` and `Master` are "god "objects in order to allow components to easily
  gain access to dependencies they might need by just holding a single reference.  While
  this is exactly the sort of pattern I'm aiming to eliminate, there are a few places
  which will need significant redesign to accomplish this.  The FATE subsystem within the master
  is one of these.  Currently instances of `MasterRepo` act on the object graph entirely
  by means of references obtained via dereference of `Master`, and this is by design.
  Allowing instances of `MasterRepo` to gain access to appropriate portions of the
  object graph without having a god object will be tricky.

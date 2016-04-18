..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Stochastic Weighing Scheduler
=============================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/stochastic-weighing-scheduler

The filter scheduler is the de-facto standard scheduler for Cinder and has
a lot of desirable properties. However there are certain scenarios where it's
hard or impossible to get it to do the right thing. I think some small tweaks
could help make admin's lives easier.


Problem description
===================

I'm concerned about 2 specific problems:

1. When there are multiple backends that are not identical, it can be hard to
   ensure that load is spread across all the backends. Consider the case of a
   few "large" backends mixed with some "small" backends. Even if they're from
   the same vendor by default new volumes will go exclusively to the large
   backends until free space decreases to the same level as the small backends.
   This can be worked around by using something other than free space to weigh
   hosts, but no matter what you choose, you'll have a similar issue whenever
   the backends aren't homogeneous.

2. Even if the admin is able to ensure that all the backends are identical in
   every way, at some point the cloud will probably need to grow, by adding
   new storage backends. When this happens there will be a mix of brand new
   empty backends and mostly full backends. No matter what kind of weighing
   function you use, initially 100% of new requests will be scheduled on the
   new backends. Depending on how good or bad the weighing function is, it
   could take a long time before the old backends start receiving new requests
   and during this period system performance is likely to drop dramatically.
   The problem is particularly bad if the upgrade is a small one: consider
   adding 1 new backend to a system with 10 existing backends. If 100% of
   new volumes go to the new backend, then for some period, there will be 10x
   load on the single backend.

There is one existing partial solution to the above problems -- the goodness
weigher -- but that has some limitations worth mentioning. Consider an ideal
goodness function -- an oracle that always returns the right value such
that the best backend for new volumes is sorted to the top. Because the inputs
to the goodness function (other than available space) are only evaluated every
60 seconds, bursts of creation requests will nearly always go to the same
backend within a 60 second window. While we could shrink the time window of
this problem by sending more frequent updates, that has its own costs and also
has diminishing returns. In the more realistic case of a goodness function
that's non-ideal, it may take longer than 60 seconds for the goodness function
output to reflect changes based on recent creation requests.


Use Cases
=========

The existing scheduler handles homogeneous backends best, and relies on a
relatively low rate of creation requests compared to the capacity of the whole
system, so that it can get keep up to date information with which to make
optimal decisions. It also deals best with cases when you don't add capacity
over time.

I'm interested in making the scheduler perform well across a broad range of
deployment scenarios:

1. Mixed vendor scenarios
2. A mix of generations of hardware from a single vendor
3. A mix of capacities of hardware (small vs. large configs)
4. Adding new capacity to a running cloud to deal with growth

These are all deployer/administrator concerns. Part of the proposed solution
is to enable certain things which are impossible today, but mostly the goal
is to make the average case "just work" so that administrators don't have to
babysit the system to get reasonable behavior.


Proposed change
===============

Currently the filter scheduler does 3 things:

1. Takes a list of all pools and filters out the ones that are unsuitable for
   a given creation request.
2. Generates a weight for each pool based on one of the available weighers.
3. Sorts the pools and chooses the one with the highest weight.

I call the above system "winner-take-all" because whether the top 2 weights
are 100 and 0 or 49 and 51, the winner gets the request 100% of the
time.

I propose adding a new option to the filter scheduler called
"weighted_stochastic". It should default to False, which would give the
current winner-take-all behavior. If set to True however, the scheduling
algorithm would change as follows:

In step 3 above, rather than simply selecting the highest weight, the
scheduler would sum up the weight of all choices, assign each pool a subset of
that range with a size equal to that host's weight, then generate a random
number across the whole range and choose the pool mapped to that range.

An easier way to visualize the above algorithm is to imagine a raffle drawing.
Each pool is given a number of raffle tickets equal to the pool's weight
(assume weights normalized from 0-100). The winning pool is chosen by a raffle
drawing. Every creation request results in a new raffle being held.

Pools with higher weights get more raffle tickets and thus have a higher
chance to win, but any pool with a weight higher than 0 has some chance to
win.

The advantage to the above algorithm is that it distinguishes between weights
that are close (49 and 51) vs weights that are far (0 and 100) so just because
one pools is slightly better than another pool, it doesn't always win. Also,
it can give different results within a 60 second window of time when the
inputs to the weigher aren't updated, significantly decreasing the pain of
slow volume stats updates.

It should be pointed out that this algorithm not only requires that weights
are properly normalized (the current goodness weigher also requires this) but
that the weight should be roughly linear across the range of possible values.
Any deviation from linear "goodness" can result in bad decisions being made,
due to the randomness inherent in this approach.


Alternatives
------------

There aren't many good options to deal with the problem of bursty requests
relative to the update frequency of volume stats. You can update stats faster
but there's a limit. The limit is to have the scheduler synchronously request
absolute latest volume stats from every backend for every request. Clearly
that approach won't scale.

To deal with the heterogeneous backends problem, we have the goodness
function, but it's challenging to pick a goodness function that yields
acceptable results across all types of variation in backends. This proposal
keeps the goodness function and builds upon it to both make it stronger, and
also more tolerant to imperfection.


Data model impact
-----------------

No database changes.


REST API impact
---------------

No REST API changes.


Security impact
---------------

No security impact.


Notifications impact
--------------------

No notification impact.


Other end user impact
---------------------

End users may indirectly experience better (or conceivably worse) scheduling
choices made by the modified scheduler.


Performance Impact
------------------

No performance impact. In fact this approach is proposed expressly because
alternative solutions would have a performance impact and I want to avoid
that.


Other deployer impact
---------------------

I propose a single new config option for the scheduler. The default value for
this option is to act like the existing scheduler does. An administrator would
need to intentionally enable to option to observe changed behavior.


Developer impact
----------------

Developers wouldn't be directly impacted, but anyone working on goodness
functions or other weighers would need to be aware of the linearity
requirement for getting good behavior out of this new scheduler mode.

In order to avoid accidentally feeding nonlinear goodness values into the
stochastic weighing scheduler, we may want to create alternatively-named
version of the various weights or weighers, forcing driver authors to
explicitly opt-in to the new scheme and thus indicate that the weights
they're returning are suitably linear.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bswartz

Work Items
----------

This should be doable in a single patch.


Dependencies
============

* Filter scheduler (cinder)
* Goodness weigher (cinder)


Testing
=======

Testing this feature will require a multibackend configuration (otherwise
scheduling is just a no-op).

Because randomness is inherently required for the correctness of the
algorithm, it will be challenging to write automated functional test cases
without subverting the random number generation. I propose that we rely on
unit tests to ensure correctness because it's easy to "fake" random numbers
in unit tests.


Documentation Impact
====================

Dev docs need updated to explain to driver authors what the expectations are
for goodness functions.

Config ref needs to explain to deployers what the new config option does.


References
==========

No references.

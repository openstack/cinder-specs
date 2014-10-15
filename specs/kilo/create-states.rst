..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Volume state enforcer
=====================

https://blueprints.launchpad.net/cinder/+spec/cinder-state-enforcer

Concurrent resource access in cinder is a problem that has caused resource
corruption when simultaneous resources are mutated on by multiple cinder
entrypoints (api and manager for example). In Icehouse there has been some
addition & usage of locks around manager functions to queue up those requests
when a resource is being simultaneous worked on by multiple
functions (this stops one of those operations from concurrently mutating the
underlying resource). Sadly this is more of a *sledgehammer* approach and
hides the symptoms of the problem and makes it non-obvious when debugging what
other requests are queued up behind the lock (or why dead-locking is
occurring, if and when it does).

To help alleviate and hopefully solve this problem we will try to attack some
of these issues in a different manner, integrating a *allowed* state transition
table into the ``create_volume`` workflow and doing *strategic* state
transitions and aborting/erroring out when these state transitions are not
allowed. In the future this will help create a concrete set of well defined
states and transitions for other workflows as well (and will make it clear
while looking at code and during debugging which transitions are allowed at the
same time and what transitions are actively occurring).

Problem description
===================

A high-level description of the problem:

* Concurrent resource mutation, bad (EOM).

More detailed description:

* Locks in cinder are being added to protect against simultaneous resource
  modification, for example in ``create_volume, attach_volume,``
  ``delete_volume, detach_volume...`` a external lock is acquired in the
  manager with name ``volume_id, f.__name__``. This has helped make the
  manager more safe to concurrent resource access but the initial goal of this
  was for it to only be a temporary solution to a wider problem. One of the
  issues with this mechanism is that it is not using a `DLM`_ (distributed
  lock manager) but only a local filesystem lock instead. This means that a
  cinder-api service can mutate the resource (or initiate a request to do
  this) while a second mutation is actively in flight. When a single manager is
  active this will work out (since one of the in flight requests will backup
  behind the external lock). This solves the problem when a
  single *master* manager is running; yet this is an atypical deployment
  pattern and should not be recommended as the way to deploy and run
  cinder (it should be horizontally scalable so that there can be X
  active managers, where X is > 1). We need some other type of solution that
  scales horizontally but also solves the same end goal (disallowing
  simultaneous resource mutation by X entities at the same time).

Since the scope of this problem is bigger (it applies to all/most operations
that act on resources) we have to start somewhere so we will start by working
through how this will look for the ``create_volume`` workflow. It does raise a
larger question of how can this change be done in a *piecemeal* fashion since
the other operations will still be lock dependent, and mixing state transitions
and lock acquisition techniques will likely not end in a correct solution. We
will have to explore how to do this in a way that is *piecemeal* but also does
not destabilize cinder more.

Proposed change
===============

Instead of acquiring local filesystem locks in the manager processes refactor
the concept of a lock to instead be a set of allowed and disallowed state
transitions (which is in concept similar to the internal mechanism that a
lock uses anyway).

Lets take an abbreviated example of how this could work:

When a volume is requested to be created, a database record is created for
this volume, in this database record there exists a field called
``status`` (in fact there exists multiple of these statuses fields, in the
future these should maybe be removed?) that is used to report back to the user
data about the status of the create volume request as it moves through the
various components in cinder (api, scheduler, and manager).

This status itself has a expected transition diagram and itself is a starting
point in determining the larger states transitions that a cinder volume create
request goes through (and is allowed to go through). Instead of overriding
this ``status`` field this proposal proposes to augment the data storage layer
in cinder with a new ``resource_states`` table. It may be represented by
something other than a table depending on where this data is stored (if
`zookeeper`_ was used it would be represented as a resource tree), the only
constraint that we *must* enforce is that we can atomically fetch and update
the given state of a resource in a single atomic operation.

A potential schema could look like the following:

+--------------------------------------+----------------+---------------------+
| **Resource**                         | **State**      | **Transitioned on** |
+======================================+================+=====================+
| 7c92ee46-7a2e-4183-99c5-909f3d46a90e |  CREATING_DB   | 2014-05-22T15       |
| 7c92ee46-7a2e-4183-99c5-909f3d46a90e |  SCHEDULING    | 2014-05-23T15       |
| 7c92ee46-7a2e-4183-99c5-909f3d46a90e |  CREATING_VOL  | 2014-05-23T15       |
| 7c92ee46-7a2e-4183-99c5-909f3d46a90e |  NULL/None     | 2014-05-24T15       |
+--------------------------------------+----------------+---------------------+

This table structure will then be used (with ``NULL`` states to delimit when
a request has fulfilled its set of allowed state transitions) to determine at
the API level (before a request has been accepted) what a resource is currently
being used for and the API server can then attempt to initiate a transition to
a desired state (for example, ``DETACHING``) and depending on if this
transition is allowed (by looking at the last known state) it may fail or
succeed at performing this transition. If it succeeds it continues with the
rest of the workflow for the desired operation (subsequent transitions will
also be made in the rest of the workflow, as needed, with the final transition
being a transition to ``NULL/None``, to denote that the operation has
completed). If the transition is disallowed/fails the API request will be
denied and the operation will not be allowed to make forward progress (in the
future this model can be relaxed to allow for simultaneous state transitions
for operations where this makes sense).

To accomplish this, in the ``create_volume`` operation there exists the usage
of taskflow, which has helped decompose the workflows that volume creation
goes through (it also makes it possible to resume from a prior state if the
process crashes). This decomposition makes it obvious (or more obvious) where
the transitions should occur and what the transitions are. The proposed path
is to add in new nodes into the workflow that will perform & validate
these state transitions (attempting to mutate the above resource state table)
at a granular-enough level to be useful & meaningful (the transition table also
can be useful for operators and developers attempting to determine what is
happening inside cinder). When this is combined with notifications from
taskflow about its own internal `states`_ (via `notifications`_) the ability to
decipher what is going on internally to cinder becomes very easy & provides
invaluable information to users, developers and operators using & operating
cinder.

Alternatives
------------

One possibility for avoiding the above ``resource_states`` table is to use
a `DLM`_ and use a similar approach that is being used with file locks in
cinder. The usage of it would be similar to the usage of file locks, although
there are scenarios at RPC boundaries where it would still require state
transition validation. For example when a lock is released and an async RPC
call is made there becomes the possibility for other async RPC calls to also
be active at the same time and there would require a state transition and
lock system to be used when the receivers of those RPC calls accept and perform
the requested RPC operation.

Another possible solution that does not require state transitions is to not use
async RPC calls but instead use sync RPC calls, and the sender would only
release the `DLM`_ lock it owns after it has received confirmation that the
receiver has started to process (or accepted the request). The receiver would
then acquire the lock during this period when it accepts the request, ensuring
that correct lock hand-off happens between the send and receiver. This would
require a sensitive and hard to get correct lock hand-off code
path & process (this path would need to be tested heavily to ensure
correctness).

IMHO both of these alternative methods are too fragile and do not make the
state transition process and diagram obvious to developers, operators, and
users. This lack of information impedes cinder adoption, and makes it more
difficult to recovery from (and understand) inevitable failures and
operational issues.

.. _DLM: http://en.wikipedia.org/wiki/Distributed_lock_manager

What this does not solve
------------------------

I would also like include a note to what the scope of this specification does
**not** encompass.

* It does **not** encompass cross-project resource usage and
  inconsistencies related to state transitions being done by a project using
  cinder (for example the initiation of a detach of a volume by nova will not
  be aborted early in the nova API flow, but instead will be aborted later in
  the workflow if cinder is performing other state transitions on that
  resource).
* It also does **not** also stop cinder from deleting a volume underneath
  nova (aka a VM can be using a volume while cinder is deleting it).

These are larger cross-project consistency issues and will need to be solved
at a higher level across the projects. It should be noted that once a project
itself has a consistent set of states and transitions it becomes *much* easier
to make cross-project consistency possible (without **internal** consistency
cross-project resource usage might as well be discouraged/avoided).

Data model impact
-----------------

See the above proposed table.

Cross-project impact
--------------------

We **must** be careful to retain the existing API so that nova which is
dependent on cinders currently visible states continues to work. This just
means that we need to have a exposed mapping that nova is compatible with;
while we have an internal mapping which is much more detailed and consistent.

REST API impact
---------------

Maybe in the future.

Security impact
---------------

N/A

Notifications impact
--------------------

None currently, the state transition information could also be sent out to
the notification system if this is desirable in the future to do so.

Other end user impact
---------------------

End users should now expect more errors (or try again later) responses when
performing operations concurrently on the same set of resources. Previously
some of these operations may or may not have succeeded.

Performance Impact
------------------

A new table will be created in `sqlalchemy`_ and a new model will be created
for this new schema. This table will be high read and write traffic (since all
operations that occur in cinder will write data to it) so it might be
recommended to alter the table type to a more friendly format that performs
better for this tables limited usage. Since this table is relatively simple it
should also be possible in the future (when correctness is achieved) to
switch this table to some other backend that can optimize itself for small
read/writes with little history (history is not as useful, except for operators
and developers who wish to interrogate what has happened to a
resource in the past).

.. _sqlalchemy: http://www.sqlalchemy.org/

Other deployer impact
---------------------

N/A

Developer impact
----------------

Developers would likely get a lot of the benefit of this information to start
since it will help them understand the states a workflow goes through (at
the cinder level), combining this with the event stream that taskflow emits
creates a lot of useful runtime information that can be used while running
cinder or while developing cinder (where to add new state transitions in
becomes more obvious when the state transitions that occur are well defined
and understood).

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Harlowja

Other contributors:

* DuncanT
* Others?
* You the person reading this?

Work Items
----------

* Determine state digram and debate what states should be used internally to
  cinder (the **critical** must-have states) and what states are more
  **informational** (DuncanT has apparently done some of this analysis).
* Create database schema migration/addition for the decided upon new schema.
* Create database models for new schema (and determine and discuss on how the
  atomic state update will be accomplished).
* Identify key locations where these state transitions will occur (before or
  after which taskflow tasks) or at a layer outside of taskflow.
* Add new tests that trigger these new state transitions and violation checks,
  ensuring that what is desired to occur actually occurs.
* Simultaneously work on creating a model inside of taskflow that can help
  other projects avoid recreating chunks of the above code for there own
  similar needs/use-cases.
* Test like *crazy*.

  * Do load-testing/concurrency-testing (using rally or tempest) to verify the
    improvement has helped and not hurt cinder.

Milestones
----------

J/3 into K (this is likely not a short-term specification).

Dependencies
============

N/A

Testing
=======

Since this change affects how cinder operates at a low level, it will require
a good amount of testing to verify that concurrent operations are disallowed.
Currently tempest may not be the best way to test these concurrent operations
since to my knowledge it does not run in parallel (and only when it runs in
a controlled parallel process can u find these concurrency issues). So the
way to test these concurrency issues needs to be determined (is `rally`_ the
way to go here, using its concurrent scenarios to probe that this
feature works?).

.. _rally: https://wiki.openstack.org/wiki/Rally

Documentation Impact
====================

There may be new documentation required to explain why operations that were
allowed to occur concurrently are no longer allowed to occur concurrently since
this new state transition will be more strict as to what can and what can not
occur at the same time.

It will also become possible to start to form documents like taskflow
`states`_ that show exactly what the internals of cinder are doing
and what the allowed state transitions (aka the cinder reference operation
states) are.

References
==========

**Summit discussion/session:**

https://etherpad.openstack.org/p/juno-cinder-state-and-workflow-management

.. _states: http://docs.openstack.org/developer/taskflow/states.html
.. _zookeeper: http://zookeeper.apache.org/
.. _notifications: http://docs.openstack.org/developer/taskflow/notifications.html

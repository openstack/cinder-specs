..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Cinder Volume Active/Active support - Manager Local Locks
=============================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion.

One of the reasons for this is that we have multiple **local locks** in the
Volume nodes for mutual exclusion of specific operations on the same resource.
These locks need to be shared between nodes of the same cluster, removed or
replaced with DB operations.

This spec proposes a different mechanism to replace these local locks.


Problem description
===================

We have locks in Volume nodes to prevent things like deleting a volume that is
being used to create another volume, or attaching a volume that is already
being attached.

Unfortunately these locks are local to the nodes, which works if we only
support Active/Passive configurations, but doesn't on Active/Active
configurations when we have more than one node, since locks will not guarantee
mutual exclusion of operations on other nodes.

We have different locks for different purposes but we will be using the same
mechanism to allow them to handle an Active/Active clusters.

List of locks are:

- ${VOL_UUID}
- ${VOL_UUID}-delete_volume
- ${VOL_UUID}-detach_volume
- ${SNAPSHOT_UUID}-delete_snapshot

We are adding an abstraction layer to our locking methods in Cinder -for the
manager and drivers- using Tooz_ as explained in `Tooz locks for A/A`_ that
will use local file locks by default and will use a DLM for Active-Active
configurations.

But there are cases where drivers don't need distributed locks to work, they
may just need local locks and we would be forcing them to install a DLM just
for these few locks in the manager, which can be considered a bit extreme.


Use Cases
=========

Operators that have hard requirements, SLA or other reasons, to have their
cloud operational at all times or have higher throughput requirements will want
to have the possibility to configure their deployments with an Active/Active
configuration and have the same behavior they currently have.  But they don't
want to have to install a DLM when they are using drivers that don't require
distributed locks to operate in Active-Active mode just for 4 locks.

Also in the context of the effort to make Cinder capable of working as an SDS
outside of OpenStack, where we no longer can make a hard requirement the
presence of a DLM as we can within OpenStack, it makes even more sense for
Cinder to be able to work without a DLM present for Active-Active
configurations if the storage backend drivers don't require distributed locking
in Active-Active environments.


Proposed change
===============

This spec suggests modifying behavior introduced by `Tooz locks for A/A`_
for the case where the drivers don't need distributed locks.  So we would use
local file locks in the drivers (if they use any) and for the locks in the
manager we would use a locking mechanism based on the ``workers`` table that
was introduced by the `HA A/A Cleanup specs`_.

This new locking mechanism would be an hybrid between local and distributed
locks.

By using a locking mechanism similar to the one that local file locks provide
we'll be able to mimic the same behavior we have:

- Assure mutual exclusion between different nodes of the same cluster and in
  the node itself.
- Request queuing.
- Lock release on node crash.
- Require no additional software in the system.
- Require no operator specific configuration.

To assure the mutual exclusion using the ``workers`` table we will need to add
a new field called ``lock_name`` that will store the current operation (method
name in most cases since table already has the resource type and UUID) that is
being performed on the Volume's manager and it will be used for locking.

New locking mechanism will use added ``lock_name`` field to check if
``workers`` table already has an entry for that ``lock_name`` on the specific
resource and cluster of the node, in which case the lock is *acquired* and we
have to wait and retry later until the lock has been *released* -row has been
deleted- and we can insert the row ourselves to *acquire* the lock. This means
that in the case of attaching a volume we will only proceed with the attach if
there is no DB entry for our cluster for the volume ID with operation
``volume_attach``.

To assure mutual exclusion, this lock checking and acquisition needs to be
atomic, so we'll be using a conditional insert of the "lock" in the same way we
are doing conditional updates (compare-and-swap) to remove `API races`_.
Insert query will be conditioned to the absence of a row with some specific
restrictions, in this case conditions will be the ``lock_name``, the
``cluster`` and the ``type``.  If we couldn't insert the row acquiring the lock
failed and we have to wait, if we inserted the row successfully we have
acquired the lock and can proceed with the operation.

This process of trying to acquire, fail, wait and retry, is the exact same
mechanism we have today with the local file locks.  ``synchronize`` method when
using external synchronization will try to acquire the file lock with a
relatively small timeout and if it fails it will try again until lock is
acquired.

So by mimicking the same behavior as the file locks we are preserving the
queuing of operations we currently have, and not altering our API's behavior
and "implicit API contract" that some external scripts may be relying on.

Since we will be using the DB for the locking operators will not need to add
any software to their systems or carry out specific system configurations.

It is important to notice that timeouts for these new locks will be handled by
the cleanup process itself, as rows are removed from the DB when heartbeats are
no longer being received, thus releasing the locks and preventing a resource
from being stuck.

On a closer look at these 4 locks mentioned before we can classify them in 2
categories:

- Locks for the resource of the operation.

  - *${VOL_UUID}-detach_volume*  -  Used in ``detach_volume`` to prevent
    multiple simultaneous detaches

  - *${VOL_UUID}*  -  Used in ``attach_volume`` to prevent multiple
    simultaneous attaches

- Locks that prevent deletion of the source of a volume creation (they are
  created by ``create_volume`` method):

  - *${VOL_UUID}-delete_volume*  -  Used in ``delete_volume``
  - *${SNAPSHOT_UUID}-delete_snapshot*  -  Used in ``delete_snapshot``

For locks on the resource of the operation -attach and detach- the row in the
DB is already been inserted by the cleanup method, so we'll reuse that same row
and condition the writing of the ``lock_name`` field to the lock being
available.

As for locks preventing deletion we will need to add the row ourselves since
cleanup was not adding a row in the ``workers`` table for those resources as
they didn't require any cleanup.


Alternatives
------------

We could use a DLM, which is a stand-in replacement for local locks, but there
have been operator that have expressed their concern on adding this burden -to
their systems and duties- because they are using drivers that don't require
locks for Active-Active and would prefer to avoid adding a DLM to their
systems.

Instead of using the new locking mechanism for locks that prevent deletion of
resources we could add a filter to the conditional update -the one being used
to prevent `API Races`_- that will prevent us from deleting a volume or a
snapshot that is being used as the source for a volume adding also the
appropriate response error when we try to delete such a volume/snapshot.


Data model impact
-----------------

Adds a new string field called ``lock_name`` to the ``workers`` table.

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Small, but necessary, performance impact from changing local file locks to DB
calls.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  Anyone is welcome to help

Work Items
----------

- Add ``lock_name`` field to ``workers`` table.

- Modify Cinder's new locking methods/decorators to handle hybrid behavior.


Dependencies
============

Cleanup for HA A/A: https://review.openstack.org/236977
 - We need the new ``workers`` table and the cleanup mechanism.

Removing API Races: https://review.openstack.org/207101/
 - We need compare-and-swap mechanism on volume and snapshot deletion to be in
   place so we can add required filters.

Testing
=======

Unittests for new locking mechanism.


Documentation Impact
====================

This needs to be properly documented, as this locking mechanism will *not* be
appropriate for all drivers.


References
==========

General Description for HA A/A: https://review.openstack.org/#/c/232599/

Cleanup for HA A/A: https://review.openstack.org/236977

Removal of API Races: https://review.openstack.org/207101/


.. _HA A/A Cleanup specs: https://review.openstack.org/236977
.. _API Races: https://review.openstack.org/207101/
.. _Tooz: http://docs.openstack.org/developer/tooz/
.. _Tooz locks for A/A: https://review.openstack.org/202615/

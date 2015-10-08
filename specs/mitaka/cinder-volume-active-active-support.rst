..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Cinder Volume Active/Active support - General description
=============================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion and
this spec proposes a possible path to support Active/Active configurations in
Cinder Volumes.

This spec will only provide a general description of the problem enumerating
the different issues we have to resolve without actually going into too much
detail.  It's more an eagle's eye kind of view of the problem.

Each specific issue will have its own spec that gives a detailed description of
the problem with proposed solution to the problem.


Problem description
===================

Right now cinder-volume service only accepts Active/Passive High Availability
configurations, and there are a number of things that need to change for it to
support Active/Active configurations.

API Races
---------

On API nodes given current code we are open to races in the code that affect
resources on the database, and this will be exacerbated when working with
Active/Active configurations.

Manager Local Locks
-------------------

We have multiple local locks in the manager code of the volume nodes to prevent
multiple green threads from accessing the same resource on specific operations.

This locking is local to the nodes and doesn't extend to other nodes, so we
need to solve mutual exclusion among volume nodes of the same cluster.

Job distribution
----------------

Cinder has no concept of clusters, only has the concept of hosts and each host
implements a specific backend/service.  A mechanism is needed to group hosts
from the same cluster under the same conceptual unit while retaining the
individual identities of the nodes belonging to the cluster for differentiation
in the clean up of crashed nodes.

Cleanup
-------

Right now only one node can work on a specific backend, and therefore on the
resources that it contains, so the cleanup is done by the node itself on
startup. And if the node does not come up and the resources are left on a stale
state it is not a big deal.

It is different with an Active/Active deployment since multiple nodes are
sharing the same storage back-end and a node can only do cleanup for the nodes
he was working on when he died/failed.

It is also important to do proper cleanup even when a specific node does not
come back to life, since other nodes from the same cluster can still manage
those resources.

Data Corruption Prevention
--------------------------

Since multiple nodes will be accessing the same storage back-end we have to be
extra careful not to access resources that are accessed by other nodes.

More relevant case is when we lose connection to the DB and we no longer can
send Service Heartbeats, since Scheduler's cleanup process (explained in
Cleanup proposed changes) will come into place and we could have 2 different
nodes accessing the same resource, one because it's still working on it and the
other because it is trying to do the cleanup.

Drivers' Locks
--------------

Some drivers require mutual exclusion for certain operations or when accessing
the same resources.

This mutual exclusion is currently being done using local locks in the same way
the manager does and they need to be able to work when multiple nodes are
accessing the same storage back-end.


Use Cases
=========

Operators that have hard requirements, SLA or other reasons, to have their
cloud operational at all times or have higher throughput requirements will want
to have the possibility to configure their deployments with an Active/Active
configuration.


Proposed change
===============

API Races
---------

Races on the API nodes will be removed used compare-and-swap updates to the DB.

- Specs: https://review.openstack.org/207101/

Job distribution
----------------

Job distribution will add the concept of cluster to cinder and send jobs using
a topic message queue using the cluster instead of the host like we are doing
now.

- Specs: https://review.openstack.org/232595

Cleanup
-------

Cleanup will keep track of resources that are have ongoing operations and will
have cleanup mechanisms on the Scheduler as well as the Volume nodes.

Cleanup on the nodes will happen on initialization as it is doing now but we'll
also have an automatic cleanup job on the scheduler for the cases where a node
with the same host name is not brought up.

Automatic cleanup mechanism will be disabled by default and it will be possible
to trigger it manually.

- Specs: https://review.openstack.org/236977

Data Corruption Prevention
--------------------------

Stop listening to new jobs from the Message Broker and halt all ongoing
operations so we are no longer accessing resources on the Storage Backend.

- Specs: https://review.openstack.org/237076

Manager Local Locks
-------------------

Default solution will be using a DLM with TooZ as the abstraction layer:

- Specs: https://review.openstack.org/202615

An alternative solution, that will be initially left as nice to have, will be
available for systems that don't want to install a DLM solution and are using
drivers that don't require distributed locking for Active-Active
configurations.  This solution replaces local file locks on c-vol's manager
with a DB locking mechanism using ``workers`` DB table (introduced by Cleanup
changes).

- Specs: https://review.openstack.org/237602

Drivers' Locks
--------------

We will be using a DLM solution with TooZ as the abstraction layer:

- Specs: https://review.openstack.org/202615

Alternatives
------------

There are quite a number of alternatives to not only each of the issues we need
to fix, and they are discussed in the respective specs except for the Drivers'
lock alternative that creates a generic locking mechanism extending the locking
mechanism implemented to remove `Manager Local Locks`_.

- Specs: https://review.openstack.org/237604


Data model impact
-----------------

Discussed in the respective specs.

REST API impact
---------------

Discussed in the respective specs.

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

Discussed in the respective specs.

Other deployer impact
---------------------

Discussed in the respective specs.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Discussed in the respective specs.

Work Items
----------

- API Races
- Job distribution
- Cleanup
- Data Corruption Prevention
- Manager Local Locks
- Drivers' Locks

Dependencies
============

None


Testing
=======

Discussed in the respective specs.


Documentation Impact
====================

Discussed in the respective specs.


References
==========

None

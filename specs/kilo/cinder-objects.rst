..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cinder Objects
==========================================

https://blueprints.launchpad.net/cinder/+spec/cinder-objects

The goal of this spec is to introduce objects into cinder.  An object is used
to bundle data with methods that can operate on them.  By implementing this
spec, we would be able to pass rich objects over RPC and decouple the
database schema implementation from SQLAlchemy, enabling support for NoSQL
databases.  Database schema independence is the biggest gain with this
proposal.


Problem description
===================

There are a few problems that exists today in cinder:

* The database implementation is directly tied to SQLAlchemy.  This limits
  support for NoSQL databases.  It is a major reason behind this proposal.
* Cinder sends ids over RPC (e.g. volume id, snapshot id, consistency group
  id, or backup id).  These ids are used to query the SQLAlchemy objects,
  resulting in a database hit.
* There are multiple places where database calls are made in cinder, and this
  could result in race conditions.


Proposed change
===============

There is an approved proposal to put nova.objects into oslo as
oslo.versionedobjects (https://review.openstack.org/#/c/127532/), which would
abstract versionable internal objects for use with RPC.  The plan is to adopt
oslo.versionedobjects and abstract volumes, snapshots, backups, consistency
groups, and quotas.

Since it will take some time before versionedobjects goes into the oslo
library, the plan is to sync up with dansmith to get an early version of it
for cinder and transition to oslo.versionedobjects when it is ready.

The proposed changes are:

* Objects will be database schema independent.  The type of database
  connection can be determined from the value of "connection" in cinder.conf,
  which could be mysql, mongodb, db2, etc.  Depending on the type of
  connection, the appropriate objects (e.g. cinder.objects.volume or
  cinder.objects.nosql.volume) would be imported when objects.register_all()
  is called.
* Objects will be sent over RPC instead of ids, reducing the additional
  database hits to query the SQLAlchemy objects.
* Objects will be versioned, allowing older objects to be detected and taking
  appropriate actions when the objects are incompatible, i.e. fail gracefully
  or emulate older behavior.
* To handle cases where the underlying volume, snapshot, backup, consistency
  group, or quota could have been changed when the object is in flight,
  additional arguments (e.g. expected_volume_state and expected_task_state)
  would be included in the object save method, which would check against the
  in-database copy of the volume, snapshot, backup, consistency group, or
  quota before updates are made and fail gracefully or appropriately sync up
  the data.

The work will be done incrementally in stages, as follows.

* Obtain an early version of oslo.versionedobjects to be used in cinder.
* Abstract volume snapshot into versionedobject.
* Abstract volume backup into versionedobject.
* Iteratively convert cinder internals to use volume snapshot object.

The goal is to have oslo.versionedobjects code base in by the end of kilo-2,
and the abstraction of a few cinder internals, e.g. volume snapshots and
backups, by that time also.

Alternatives
------------

Cinder could continue to be tightly tied to SQLAlchemy, limiting support to
only SQL databases.  Also, the existing implementation of sending ids over RPC
can be left as-is.

There are proposals for state machine/state enforcement and RPC versioning,
which would help with rolling upgrades and improve the stability of cinder.
However, you would still take an additional database hit when the volume,
snapshot, backup, or consistency group ids are used to query the SQLAlchemy
objects.

Data model impact
-----------------

None.  Cinder objects provides an abstraction layer in cinder, allowing for
database schema independence.  These objects are a replacement for SQLAlchemy
objects that are being used to represent service, volume, volume type,
snapshot, quota, backup, consistency group and consistency group snapshot
throughout cinder internals.

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

Unknown.  There might be a minor hit in performance since SQLAlchemy objects
are abstracted into cinder objects.

Other deployer impact
---------------------

None

Developer impact
----------------

Today, direct database calls are made to create, read, update, and delete
volumes, volume types, snapshots, quotas, backups, consistency groups and
consistency group snapshots, and others.  With the switch to an object model,
these calls will be made against the appropriate object methods.  It will take
some time to convert cinder internals over to the object model, so the
existing convention of direct database calls should be accepted until all
object models are in place.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thang-pham

Other contributors:
  Any help is welcome

Work Items
----------

* Obtain an early version of oslo.versionedobjects to be used in cinder.
* Abstract volume snapshot into versionedobject.
* Abstract volume backup into versionedobject.
* Iteratively convert cinder internals to use volume snapshot object.


Dependencies
============

None


Testing
=======

By introducing objects, we are not changing the end user facing API or
functionality of cinder.  We are passing objects over RPC instead of ids.
Given this, no new tempest tests will be created.  However, relevant unit
tests will be created to test the new object code base.


Documentation Impact
====================

None


References
==========

* cinder-specs: https://review.openstack.org/#/c/130044/
* oslo-specs: https://review.openstack.org/#/c/127532/
* Proof of concept: https://review.openstack.org/#/c/131873/

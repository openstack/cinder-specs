..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Snapshotting attached volumes w/o force
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/fix-snapshot-create-force

Cinder requires passing the "force" flag to a snapshot create
call to create a snapshot from a volume while it is attached
to an instance.  This is unnecessary, as snapshotting attached
volumes results in crash-consistent snapshots, which are useful,
sufficient, and one of the most common cases of how snapshots
are used in the real world.

Problem description
===================

Most users and other software that create Cinder snapshots actually
want crash-consistent snapshots of attached volumes, so making this
an exception case is not productive.  Code is written to always
use "force", and users learn that it is needed to create snapshots.

In most virtualization platforms for many years, creating
crash-consistent snapshots of online disks is not an exception case,
it is a normal operation.  It should be in Cinder too.


Use Cases
=========

* easier for end users
* less surprising snapshot API for developers

Proposed change
===============

Introduce a new microversion that no longer uses a "force" flag
to control when snapshots can be created for a volume.

This means that snapshot creation succeeds for volumes that are in the
"available" or "in-use" state.  The "force" parameter is no longer needed,
but "force=true" is accepted to reduce code changes required for users who are
currently passing this flag from their code.

Snapshot creation with "force=false" will be rejected as invalid after
this new microversion.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

* Introduce a new microversion for this change
* Snapshot creation will succeed for in-use volumes w/o force flag added.
* Passing force=True will succeed for in-use volumes as it does currently,
  but this parameter is no longer needed for this case.
* Passing force=False for snapshot creation is not allowed after the new
  microversion.  This is presumably rarely used and removing it reduces
  ambiguity about what "force=False" would mean when in-use volumes can
  be snapshotted by default.

Security impact
---------------

None

Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

Minimal cinderclient changes

Performance Impact
------------------

* Fewer snapshot create calls that return HTTP 400 resulting in the user
  issuing a second snapshot create call w/ "force" added

Other deployer impact
---------------------

Developer impact
----------------

Implementation
==============

Assignee(s)
-----------

Primary assignee:
 eharney

Work Items
----------

* Fix snapshot-create
* New API microversion
* Tempest Tests

* Look at backup-create (similar problems)

Dependencies
============

Testing
=======

* New tempest tests in cinder-tempest-plugin

Documentation Impact
====================

* Minimal


References
==========


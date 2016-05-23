..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Parameter combinations for delete (vol, snap, etc.)
===================================================

https://blueprints.launchpad.net/cinder/+spec/volume-delete-parameters

This spec outlines how to improve our volume delete functionality
with regards to optional parameters which request non-default
behaviors.

Problem description
===================

It is not possible to combine both volume force delete and cascade delete.

It is also difficult to add more parameters to volume delete, since
they have to interact in a reasonable way with things like os-force-delete,
which is part of volume delete from a user's point of view, but not the
same API action.

Use Cases
=========

This makes it easier to delete a volume which may be in an odd state.

It also simplifies our API by passing optional parameters to delete
rather than separate action calls.  This allows us to combine the parameters
in meaningful ways (force + cascade), as well as extend the same combinations
to snapshot-delete, cg-delete, etc., without having to make X delete API
actions for each parameter. (eg. os-force-delete, os-force-delete-snapshot,
os-force-delete-cg, etc.)

Also consider if we wish to add the option later to delete an
attached volume with a "force-detach" parameter, etc.

This also should reduce the number of cases where an admin/user needs to use
reset-state operations.

Proposed change
===============

Deprecate use of os-force-delete and make "force" a parameter passed to
volume delete like "cascade" is.

The ability to default "force" to be an admin-only operation via config
will be maintained.

Alternatives
------------

Change nothing.

Data model impact
-----------------

None

REST API impact
---------------

* New boolean "force" parameter for volume delete,
  which defaults to False.

  This will behave the same as os-force-delete if not
  combined with other arguments.

  If combined with cascade, a cascade delete which
  ignores the volume and snapshot states will be performed.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

$ cinder delete --force --cascade <volume>

will be accepted.  This gives a "delete this volume regardless
of the state of things" operation which does not exist today.

Performance Impact
------------------

None

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
  eharney

Work Items
----------

* Add "force" as a parameter to volume delete API
* Add logic to handle combination of force and cascade
* (eventually) remove os-force-delete with a new API microversion
* Look at what to do, if anything, in this same regard for
  "unmanage", e.g., "unmanage --cascade".


Dependencies
============

None


Testing
=======

New tempest test for volume delete which uses the parameterized
version rather than os-force-delete.

Tempest test for cascade + force volume delete.


Documentation Impact
====================

New arguments for cinderclient volume delete.


References
==========

* Cascade delete
  https://review.openstack.org/#/c/201748/


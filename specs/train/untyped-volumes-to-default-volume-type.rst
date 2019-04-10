..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Untyped volumes to default volume type
======================================

https://blueprints.launchpad.net/cinder/+spec/untyped-volumes-default-volume-type

This blueprint proposes to use a default volume type instead of allowing users
to create untyped volumes.

Problem description
===================

Currently a user is able to create a volume without any volume type, since
a volume's characteristics are defined by a volume type, creating untyped
volumes shouldn't be allowed.

Use Cases
=========

Most users use volume types, and our code is simpler and less buggy if we
just always attach volume types to volumes, so we should just force all
deployments to use volume types.

Proposed change
===============

This spec proposes the following changes :

* Create a default volume type during cinder database migration or on cinder
  services start time. The default volume type will have no extra specs
  assigned to it and will be named ``__DEFAULT__``
* If a volume type named ``__DEFAULT__`` already exists, the deployer
  needs to rename or remove it before upgrading
* Add an online migration to convert all untyped volumes and snapshots to
  ``__DEFAULT__`` volume type
* Set a default value ``__DEFAULT__`` for ``default_volume_type`` config
  option so the default value is picked when it is unset in cinder.conf
* Don't allow deletion of the ``__DEFAULT__`` volume type via type-delete
* Updating of volume type (``__DEFAULT__``) will be handled by MANAGE_POLICY
  which defaults to admin-only

Alternatives
------------

1. Mandate specifying volume type while creating volumes.
2. Do this as a behind-the-scenes DB migration rather than relying on manual
   intervention with upgrade checkers.
3. Do this as a best-effort-if-it's-safe operation in the volume manager,
   which would migrate most deployments, and skip those that are ruled out for whatever reason.

REST API impact
---------------

None

Data model impact
-----------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* All volumes created without specifying the volume-type parameter
  will be associated with the default volume type.
* Untyped Volumes and Snapshots will be assigned ``__DEFAULT__``
  volume type
* Users will no longer be able to create a volume with None volume type.

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
  Rajat Dhasmana <rajatdhasmana@gmail.com>

Other assignee:
  Eric Harney <eharney@redhat.com>

Work Items
----------
By restricting untyped volumes, we need to do the following changes:

* Add an upgrade check to verify that current deployment doesn't contain
  any volume type named ``__DEFAULT__`` else the upgrade will fail

* Create ``__DEFAULT__`` volume type at the time of cinder db migration or
  service start time

* Add upgrade check to verify no type named ``__DEFAULT__`` exists before
  upgrading

* Add an online migration to convert all untyped volumes and snapshots to
  ``__DEFAULT__`` volume type

* Related code changes to associate default_volume_type to volumes if no
  volume type is specified by user

Dependencies
============

None

Testing
=======

Unit-tests, tempest and other related tests will be implemented.

Documentation Impact
====================

Need to update volume type related docs.

References
==========

None

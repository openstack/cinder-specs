..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Temporary Resource Tracking
===========================

https://blueprints.launchpad.net/cinder/+spec/temp-resources

Improve Cinder's temporary resource tracking to prevent related quota issues.


Problem description
===================

Cinder doesn't currently have a consistent way of tracking temporary resources,
which leads to quota bugs.

In some cases temporary volumes use the ``temporary`` key in the admin metadata
table to mark them and in other cases we determine a volume is temporary based
on its ``migration_status`` field, there are even cases where volumes are not
being marked as temporary.  Due to this roundabout way of marking temporary
volumes and having multiple options makes our Cinder code error prone, as is
clear by the number of bugs around it.

As for temporary snapshots, Cinder doesn't currently have any way of reliably
tracking them, so the code creating temporary resources assumes that everything
will run smoothly and the deletion code in the method will be called after
successfully completing the operation.  Sometimes that is not true, as the
operation could fail and leave the temporary resource behind, forcing users to
delete them manually, which messes up the quota, since the REST API delete call
doesn't know it shouldn't touch the quota.

When we say that we don't have a reliable way of tracking snapshots we refer to
the fact that even though snapshots have a name that helps identify them, such
as ``[revert] volume %s backup snapshot`` and ``backup-snap-%s``, these are
also valid snapshot names that a user can assign, so we cannot rely on them to
differentiate temporary snapshots.


Use Cases
=========

There are several cases where this feature will be useful:

* Revert to snapshot is configured to use a temporary snapshot, but either the
  revert fails or the deletion of the temporary volume fails, so the user ends
  up manually deleting the snapshot, and the quota is kept in sync with
  reality.

* Creating a backup of an in-use volume when ``backup_use_temp_snapshot`` is
  enabled fails, or the deletion of the temporary resource failed, forcing the
  user to manually deleting the snapshot, and the user wants the quota to be
  kept in sync with reality.

* A driver may have some slow code that gets triggered when cloning or creating
  a snapshot for performance reasons but that would not be reasonable to
  execute for temporary volumes.  An example would be the flattening of cloned
  volumes on the RBD driver.


Proposed change
===============

The proposed solution is to have an explicit DB field that indicates whether a
resource should be counted towards quota or not.

The field would be named ``use_quota`` and it would be added to the ``volumes``
and ``snapshots`` DB tables.  We currently don't have temporary backups, so no
field would be added to the ``backups`` DB table.

This would replace the ``temporary`` admin metadata entry and the
``migration_status`` entry in 2 cycles, since we need to keep supporting
rolling upgrades where we could be running code that doesn't know about the new
``use_quota`` field.

Alternatives
------------

An alternative solution would be to use the ``temporary`` key in the volumes'
admin metadata table like we are doing in some case and create one such table
for snapshots as well.

With that alternative DB queries could become more complex, unlike with the
proposed solution where they would become simpler.

Data model impact
-----------------

Adds a ``use_quota`` DB field of type ``Boolean`` to both ``volumes`` and
``snapshots`` tables.

It will have an online data migration to set the ``use_quota`` field for
existing volumes as well as an updated ``save`` method for ``Volume`` and
``Snapshot`` OVOs that sets this field whenever they are saved.

REST API impact
---------------

There won't be any new REST API endpoint since the ``use_quota`` field is an
internal field and we don't want users or administrators modifying it.

But since this is useful information we will add this field to the volume's
JSON response for all endpoints that return it, although with a more user
oriented name ``consumes_quota``:

* Create volume

* Show volume

* Update volume

* List detailed volumes

* Create snapshot

* Show snapshot

* Update snapshot

* List detailed snapshots

Security impact
---------------

None.

Active/Active HA impact
-----------------------

None, since this mostly just affects whether quota code is called or not when
receiving REST API delete requests.

Notifications impact
--------------------

None.

Other end user impact
---------------------

The change requires a patch on the python-cinderclient to show the new returned
field ``consumes_quota``.

Performance Impact
------------------

There should be no performance detriment with this change, since the field
would be added at creation time and would not require additional DB queries.

Moreover performance improvements should be possible in the future once we
remove compatibility code with the current temporary volume checks, for example
not requiring writing to the admin metadata table, making quota sync
calculations directly on the DB, etc.

Other deployer impact
---------------------

None.

Developer impact
----------------

By default Volume and Snapshot OVOs will use quota on creation (set
``use_quota`` to ``True``) and when developers want to create temporary
resources that don't consume quota on creation or release it on deletion will
need to pass ``use_quota=False`` at creation time.

Also when doing quota (adding or removing) new code will have to check this
field in Volumes and Snapshots.

It will no longer be necessary to add additional admin metadata or check the
``migration_status``, which should make it coding easier and reduce the number
of related bugs.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Work Items
----------

* DB schema changes.

* DB online migration and OVO changes.

* Update existing operations that mark volumes as temporary to use the new
  ``use_quota`` field.

* Update operations that are not currently marking resources as temporary to do
  so with the new ``use_quota`` field.

* REST API changes to return the ``use_quota`` field as ``consumes_quota``.

* Cinderclient changes.


Dependencies
============

None.


Testing
=======

No new tempest test will be added, since the case we want to fix is mostly
around error situations that we cannot force in tempest.

Unit tests will be provided as with any other patch.


Documentation Impact
====================

The API reference documentation will be updated.


References
==========

Proposed Cinder code implementation:

* https://review.opendev.org/c/openstack/cinder/+/786385

* https://review.opendev.org/c/openstack/cinder/+/786386

Proposed python-cinderclient code implementation:

* https://review.opendev.org/c/openstack/python-cinderclient/+/787407

Proposed code to leverage this new functionality in the RBD driver to not
flatten temporary resources:

* https://review.opendev.org/c/openstack/cinder/+/790492

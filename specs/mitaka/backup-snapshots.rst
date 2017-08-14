..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Backup Snapshots
================

https://blueprints.launchpad.net/cinder/+spec/backup-snapshots

Currently we can backup a volume, but not a snapshot. This spec proposes to
provide a way to backup snapshots.

Problem description
===================

Today we can backup a volume, but we cannot directly backup a snapshot. The
ability to backup a volume directly is valuable because it allows a volume to
be backed up in one step. Also volume backup has been available ever since
backup was introduced in Cinder. Users also take snapshots from the volumes as
a way to protect their data. These snapshots reside on the storage backend
itself. Providing a way to backup snapshots directly will allow the user to
protect the snapshots taken from the volumes on a backup device, separately
from the storage backend.

Use Cases
=========

There are users who have taken many snapshots and would like a way to protect
these snapshots. This proposal to backup snapshots provides another layer of
data protection.

There are other projects in OpenStack focusing on data protection such as
Freezer, Smaug, Raksha, etc. They are all in different stages of design,
development, or adoption. The backup API in Cinder is not a replacement of
those projects which are doing a full blown orchestration of data protection
for all resources in OpenStack (not just block storage). Instead, Cinder APIs
can be consumed by those higher level projects for data protection and can
also be used directly by users who do not need those higher level projects.

Proposed change
===============

In summary, the following changes will happen:

* A field ``snapshot_id`` will be added to the request body of the existing
  backup API.
* A new column ``snapshot_id`` will be added to the backups table.
* The backup_volume logic in driver.py and lvm.py will be changed to use the
  ``snapshot_id`` if it is not None.
* The logic to find the latest parent during the incremental backup will be
  adjusted to take into account backups from snapshots.

Note: No new driver API will be introduced in this proposal.

Steps to create a backup from snapshot are as follows by default using the
internal tenant:

* Create a temporary volume from the snapshot
* Attach the temporary volume
* Do backup from the temporary volume
* Detach the temporary volume
* Cleanup temporary volume

If the driver has implemented the attach snapshot interface introduced in the
Liberty release (see the developer impact section), backing up a snapshot will
be done using the following steps:

* Attach the snapshot
* Do backup from the snapshot
* Detach the snapshot

For incremental backups, because the latest parent is calculated automatically
by looking at the timestamps, the logic has to be changed to accommodate
backups from snapshots. For backups from snapshots, we need to look at the
timestamps of the snapshots; for backups from volumes, we still look at the
timestamps of the backups as before. The parent of an incremental backup of a
snapshot could be a backup from a previous snapshot or a backup from the
volume, depending on the timestamps when the previous snapshot was taken vs
when the volume backup was taken. The backup with the latest timestamp will be
chosen as the parent. A new column will be created to record the timestamp of
the data in the backups table. For a backup from volume, the data timestamp
field will be the same as the ``created_at`` field in the backups table. For a
backup from snapshot, the data timestamp field will be the same as the
``created_at`` field of the snapshot.

Alternatives
------------

Here is a manual alternative:

* Create a volume from the snapshot
* Backup the volume
* Delete the volume

Data model impact
-----------------

Add the following new column to the backups table for snapshot id. This field
will be null if the backup is from a volume::

    snapshot_id = Column(String(36))

Add the following new column to the backups table to record the timestamp of
the data::

    data_timestamp = Column(DateTime)

Note that the following column will still be required for a backup from
snapshot::

    volume_id = Column(String(36), nullable=False)

REST API impact
---------------

Change the existing create backup API to take a snapshot id. Either
``volume_id`` or ``snapshot_id`` has to be provided for the create backup API,
but not both. The ``snapshot_id`` is required for backing up a snapshot.

* Create backup

  * V2/<tenant id>/backups
  * Method: POST
  * JSON schema definition for V2::

        {
            "backup":
            {
                "display_name": "nightly001",  # existing
                "display_description": "Nightly backup",  # existing
                "volume_id": "xxxxxxxx",  # existing
                "snapshot_id": "xxxxxxxx",  # new
                "container": "nightlybackups",
                ......
            }
        }

Security impact
---------------

None

Notifications impact
--------------------

Currently notifications are sent out when a backup is created, restored, and
deleted. The notification data needs to be updated with the ``snapshot_id`` if
necessary.

Other end user impact
---------------------

End user will be able to create a backup from a snapshot.

Performance Impact
------------------

No obvious performance impact.

Other deployer impact
---------------------

The deployer will be able to backup a snapshot.

Developer impact
----------------

All volume drivers will get the backup from snapshot feature with this
proposal. No additional changes are required.

If a driver wants to use a more optimal way by attaching the snapshot, it can
implement the following interfaces that were added in the Liberty release to
support non-disruptive backups:

* initialize_connection_snapshot
* terminate_connection_snapshot
* create_export_snapshot
* remove_export_snapshot

The following function can also be overridden by the driver which returns
False by default:

* backup_use_temp_snapshot

Note: All of the driver APIs specified above were added in the Liberty
release. No new driver APIs are introduced by this spec.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <xing-yang>

Other contributors:
  <None>

Work Items
----------

* Make changes to the backup API to support backup snapshot.
* Make changes to the backups db table to add a ``snapshot_id`` column.
* Make changes to the ``backup_volume`` function in driver.py and lvm.py to
  support backing up a snapshot.
* Make changes to the incremental backups to take into account backups created
  from snapshots.
* Make sure the code has good comments to explain different code paths.

Dependencies
============

None

Testing
=======

Unit tests and tempest tests will be provided.

Documentation Impact
====================

Documentation will be modified to describe how to use this feature. We will
make sure both the existing use cases and the new use cases are clearly
documented to avoid any confusion. The following should be covered:

* Do a full backup of a volume with status being 'available' or 'in-use'.
* Do an incremental backup of a volume with status being 'available' or
  'in-use'.
* Do a full backup of a snapshot.
* Do an incremental backup of a snapshot.

Developer documentation should also be created to explain how the different
backup cases are handled and how it would impact the developers working on
drivers.

References
==========
Code is submitted here:
https://review.openstack.org/#/c/243406/

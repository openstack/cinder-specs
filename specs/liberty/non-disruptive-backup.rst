..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Non Disruptive Backup
==========================================

https://blueprints.launchpad.net/cinder/+spec/non-disruptive-backup

Currently a backup operation can only be performed when a volume is
detached. This blueprint proposes to use a temporary snapshot or
volume to do non-disruptive backup.

Problem description
===================

If a volume is attached, it has to be detached first before it can
be backed up. This is not efficient and it disrupts normal operations
by cloud users just to do a backup.

Use Cases
=========

The use case is that an end user would like to do a backup without
detaching the volume.

Proposed change
===============

For an attached volume, taking a temporary snapshot is usually less
expensive than creating a whole temporary volume. If we can attach the
snapshot and read it directly, we do not have to create another
temporary volume. It is quicker to complete the backup this way.

If a driver has not implemented attach snapshot and does not have a way
to read from a snapshot, we can create a temporary volume from the
attached source volume, and backup the temporary volume.

For the LVM driver, because the backup service is on the same node as
the volume driver, we can just read the local path of the temporary
snapshot after it is created from the attached source volume. We will
create a temporary snapshot, but there is no need to attach snapshot
for LVM.

In the future, if the backup service can be on a different node than
the volume driver, then we can look into whether to do attach snapshot
for LVM as well, however, there are concerns that introducing the
attach snapshot logic will complicate the LVM code. This is for the
future, so it can be decided later.

This proposal will be implemented in multiple steps.

Step 1:

For the LVM driver, take a temporary snapshot of the attached volume
and back it up.

* Take a temporary snapshot
* Do backup from the local path of the temporary snapshot
* Cleanup temporary snapshot

For other drivers, provide a default implementation by creating a
temporary volume and backing it up.

* Create a temporary volume from the source volume
* Attach the temporary volume
* Do backup from the temporary volume
* Detach the temporary volume
* Cleanup temporary volume

Note: If the volume is not attached, it will be backed up the same
way as before.

Step 2:

Provide a more optimized way for other drivers to use a temporary
snapshot to backup an attached volume. For drivers that cannot read
from a local path of a snapshot, we need to define an interface for
attaching a snapshot, similar to attaching a volume.

* Take a temporary snapshot
* Attach the temporary snapshot
* Do backup from the temporary snapshot
* Detach the temporary snapshot
* Cleanup temporary snapshot

Step 3 (future):

If the backup service can use the volume driver on a remote node
in the future, we can provide a way for the call to happen through
RPC. This is similar to step 2, except that it will call the volume
manager on a remote node using RPC API to attach snapshot.

Alternatives
------------

There are a couple of alternatives:

* Detach the volume and back it up.

* Take a snapshot of the attached volume, create a volume from the
  snapshot and then back it up.

Data model impact
-----------------

Step 1:

Add the following new columns to the backups table for the temporary
volume and snapshot id:

temp_volume_id = Column(String(36))
temp_snapshot_id = Column(String(36))

Add the following new column to the volumes table to persist the
status of a volume before it is changed to 'backing-up' status.
This will be used to restore the volume status back to either
'available' or 'in-use' after the backup is complete. This is
also used for cleaning up failed backups when the backup service
is restarted. This is because the cleanup procedure is different
for 'available' and 'in-use' volumes.

previous_status = Column(String(255))

Step 2:

Add the following new column to the snapshots table. The snapshots
table already has provider_id and provider_location like the volumes
table. This provider_auth column is needed to save the authentication
data when a target is created for the snapshot. This is needed if we
want to attach the snapshot.

provider_auth = Column(String(255))

REST API impact
---------------

Step 1:

Change the existing create backup API to take a force flag.
If the volume is 'in-use', the force flag has to be True.
By default it is False. The force flag is not needed for
'available' volumes.

* Create backup
  * V2/<tenant id>/backups
  * Method: POST
  * JSON schema definition for V2:

    .. code-block:: python

        {
            "backup":
            {
                "display_name": "nightly001",  # existing
                "display_description": "Nightly backup",  # existing
                "volume_id": "xxxxxxxx",  # existing
                "container": "nightlybackups",
                "force": True,  # new
            }
        }

Step 2:

The following driver APIs will be added to support attach snapshot and
detach snapshot.

attach snapshot:

.. code-block:: python

 def _attach_snapshot(self, context, snapshot, properties,
                      remote=False)
 def create_export_snapshot(self, conext, snapshot)
 def initialize_connection_snapshot(self, snapshot, properties,
                                    ** kwargs)

detach snapshot:

.. code-block:: python

 def _detach_snapshot(self, context, attach_info, snapshot,
                      properties, force=False, remote=False)
 def terminate_connection_snapshot(self, snapshot, properties,
                                   ** kwargs)
 def remove_export_snapshot(self, context, snapshot)

Alternatively we can use an is_snapshot flag for volume and snapshot
to share common code without adding new functions, but it will make
the code confusing and hard to read. So there is a trade off between
reducing code duplication and increasing code readibilty here.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

End user will be able to create a backup without detaching the volume.

Performance Impact
------------------

No obvious performance impact.

If we can attach the snapshot and back it up with the proposed change,
it will be cleaner and easier than manually taking a snapshot, creating
a volume from the snapshot, and then backing it up and deleting it.

Other deployer impact
---------------------

The deployer will be able to backup an attached volume.

Developer impact
----------------

Driver developers can implement the proposed new driver APIs for
more efficient backup. This is not required though. A default
implementation will be provided to create a temporary volume from
the source volume as discussed earlier.


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

* Step 1: Provide a default implementation.
  Patch in review here: https://review.openstack.org/#/c/193937/

* Step 2: Provide a more optimal implementation by adding new driver
  APIs to support attach snapshot.
  WIP patch proposed here: https://review.openstack.org/#/c/201249/

* Step 3 (future): This will happen after the backup service is
  decoupled from the volume driver.

Dependencies
============

None


Testing
=======

Unit tests will be provided.

Documentation Impact
====================

Documentation will be modified to describe this feature.

References
==========

* Link to summit discussion:
  https://etherpad.openstack.org/p/cinder-liberty-backup-using-snapshot

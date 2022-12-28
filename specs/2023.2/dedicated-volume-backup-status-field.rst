..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Dedicated backup status field for volumes
==========================================

This spec proposes to introduce a new `backup_status` field for volumes
to remove the blocking or serialization that active backup tasks impose
on volume attachment workflows.


Problem description
===================

Currently all cinder tasks use the status field to check for suitable
volume `status` while performing any particular operation. During the active
phase of a task, its status is also held and updated via the same volume status
field. And finally also certain errors thrown by a task are communicated back
via this field. This single field in essence creates a locking or
synchronization mechanism to have only one task act on a volume at any one time.

While this is helpful for coordinating tasks affecting the volume itself,
applying the same logic to backups is actually not required or helpful:

* Actions on the volume itself (such as `resize` or `attaching`) and backups
  backups (backing-up) not techically relate to each other. The backup-driver
  and block device-driver act independently and backups are read off a volume
  snapshot or, in case the config option `backup_use_temp_snapshot` is set to
  false, a clone of the volume.

* A backup might take quite a long time to finish and is blocking any other task
  for the volume in the meantime. If we assume a 8TB volume is being
  backed up in full and even if the backup was running at 1 GB/s the
  volume backup will still take ~2.5 hrs to complete. Decoupling this from a
  state machine perspective (since it already is for most drivers or
  implementations which work via snapshots) seems to be quite beneficial.


Use Cases
=========

There are two sides to the use-cases of decoupling backup tasks from other tasks
on a volume:

1. For cloud operators - Currently detaching volume from source compute node and
attaching it to the destination compute node during an instance live-migration
is blocked by a concurrently run volume backup. To make matters worse, the
potentially long running backup (task) could also have been triggered by the
user and then be blocking administrative actions such as the live-migration of
all instances to others hosts to take a hypervisor down for maintenance.

Provided sufficiently large volumes or slow backup transfer rates this could
cause users to "lock out" administrative tasks indefinitely.

2. For cloud users - Urgent operational tasks are blocked by an active backup
task currently. A running backup blocking the users ability to quickly resize a
volume that is running low of available space or attach it to another instance.
This issue is worsened if the backup tasks is even triggered by the cloud
provider or some automatic scheduling.


Proposed change
===============

For volumes there shall be field `backup_status` to hold the backup related
values currently stored in `status`:

* 'backing-up'
* 'error_backing-up'
* 'restoring-backup'
* 'error_restoring'

(The `backup_status` certainly can also be `null`, to indicate there currently
is no backup related status for this volume.)

Those shall then be removed from the list of (valid) values of `status`
and be values for `backup_status`.

There will be some changes to the conditional checks for certain tasks to be
started, but most usually depend on those volume status values that would remain
in the status field anyways.


Alternatives
------------

There was a discussion around a spec [1] moving all task
status to a new field. This ended up being way too complex and not really
suitable for the described use-cases of decoupling volume backups.


Data model impact
-----------------

* An additional field `backup_status` would have to be added to the volume table,
  together with a change to the list of valid values it might hold.

* This will be introduced via a schema change to the database first,
  to add the new field.

* This change is then followed by an online update/upgrade to split up the
  "moved out" status related to backups in their newly dedicated fields.

* The valid values for volume status would then also have to be reduced.
  https://opendev.org/openstack/cinder/src/commit/5c23c9fbe41baef22a71eac4406fd9db269d1271/cinder/objects/fields.py#L168-L190
  As the status `backing-up`, `error_backing-up` and are only to be stored in
  the backup status.

* The method `conditional_update` needs to support different versions for the
  volume data model during the update process of the database. In addition all
  calls to the method in the cinder project have to be updated to use the new
  field.


REST API impact
---------------

Due to the introduction of a new field and the following split up of the status
field for a volume a new API microversion is required. To serve older
microversions a mapping between the two data-models has to be introduced.
In essence the API has to present the status from both fields
(status, backup_status) in the (previously used) single status field when older
microversions are used.

The additional field of a backup_status would need to be sent to the user. The
valid status of a volume would only allow the reduced set. In addition
endpoints the reset_status_backup action needs to be adapted to the new
data model to allow (admin only) setting a backup status.

**NOTE**: The list of endpoints to be changed is based on the current proposed
change and is subject to change.

* Show volume

.. code-block:: json

    {
      "volume": {
          "attachments": [],
          "availability_zone": "nova",
          "bootable": "false",
          "consistencygroup_id": null,
          "created_at": "2018-11-29T06:50:07.770785",
          "description": null,
          "encrypted": false,
          "id": "f7223234-1afc-4d19-bfa3-d19deb6235ef",
          "links": [
              {
                  "href": "http://127.0.0.1:45839/v3/89afd400-b646-4bbc-b12b-c0a4d63e5bd3/volumes/f7223234-1afc-4d19-bfa3-d19deb6235ef",
                  "rel": "self"
              },
              {
                  "href": "http://127.0.0.1:45839/89afd400-b646-4bbc-b12b-c0a4d63e5bd3/volumes/f7223234-1afc-4d19-bfa3-d19deb6235ef",
                  "rel": "bookmark"
              }
          ],
          "metadata": {},
          "migration_status": null,
          "multiattach": false,
          "name": null,
          "os-vol-host-attr:host": null,
          "os-vol-mig-status-attr:migstat": null,
          "os-vol-mig-status-attr:name_id": null,
          "os-vol-tenant-attr:tenant_id": "89afd400-b646-4bbc-b12b-c0a4d63e5bd3",
          "replication_status": null,
          "size": 10,
          "snapshot_id": null,
          "source_volid": null,
          "status": "creating",
          "backup_status": null,
          "updated_at": null,
          "user_id": "c853ca26-e8ea-4797-8a52-ee124a013d0e",
          "volume_type": "__DEFAULT__",
          "provider_id": null,
          "group_id": null,
          "service_uuid": null,
          "shared_targets": true,
          "cluster_name": null,
          "volume_type_id": "5fed9d7c-401d-46e2-8e80-f30c70cb7e1d",
          "consumes_quota": true
      }
    }


* Update volume

  * return additional field for the backup_status

* List volumes

  * show additional field for the backup_status

  * additional filter for the backup_status

* List detailed volumes

  * show additional field for the backup_status

  * additional filter for the backup_status

Additionally (only available with the use of the new micro version)

* Set volume backup_status

* Unset volume backup_status


Security impact
---------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------


Performance Impact
------------------

While not a performance impact per se, having the backup state-machine
decoupled from the status will reduce the serialization of tasks
happening for a volume.


Other deployer impact
---------------------


Developer impact
----------------

The state of a volume can only be set to the reduced list of status.
In addition the backup status can now only be set or unset via the
introducted backup_status field.

The change above has the impact that all methods have to be checked which allow
concurrent interaction with a volume and use the backup_status field instead of
the status field to indicate a running process.
This change should be communicated to other developer teams that rely on the
cinder api to check on the status to either use an old microversion or update
to use the status and the backup_status.

Because the status is currently used as a locking mechanism to prevent actions
to start if an invalid status is reached, the method calls in the api have to
be updated to also include a check for the backup_status if necessary. Some of
the work here is currently done by the conditional_update method which needs to
receive support for the updated volume model. This versioning is only needed if
an old database model is received. The API always sends the new model even if
an older API version is used due to the translation layer. This translation
layer guarantees backwards compatibility with older API versions and translates
(status, backup_status) -> status and status -> (status, backup_status).

To be able to perform an online migration of the database for an update of
openstack a method to remap the status -> (status, backup_status) and save it in
the database is necessary. This method should only be called if the schema
update is done. This method will allow older openstack versions to be able to
perform as before and set or check on their status as needed. These methods
should be removed in further releases.


Upgrade impact
--------------

none


Implementation
==============

The following existing restrictions on which actions / tasks can happen
on a volume have to be maintained with the changes implemented:

* Reject volume deletion while volume is currently being backed up
* Reject concurrent backups of a volume if one is already in progress
* Reject volume or "block storage" migration if a backup is currently running


Assignee(s)
-----------

Primary assignee:
  Christian Rohmann (IRC: crohmann)

Other contributors:


Work Items
----------

* Split up the status field to the above mentioned status and backup_status
  fields

  * Update SQL data model by adding a backup_status column to the volumes table

  * Create DB migration (Alembic)

  * additional method(s) for the model to allow an online migration of the database

  * Update volume class by adding backup_status field and bumping the version

* Introduce validation methods for the new backup_status field to guarantee that
  no wrong status can be set

* Update validation methods for the status field to guarantee that
  no wrong status can be set

* Add versioning to the conditional_update method for `old` database models

* Update method calls to conditional_update to use status and backup_status

* Introduce the API-Layer as a translator to serve older micro versions

* Introduce a new microversion to allow backups to be performed in isolation
  from other operations that use volume's status field as a locking mechanism.

* Documentation

  * Breaking changes

  * API Documentation

    * Models

    * Endpoints

  * Upgrade guide/scripts for older database models

    * online migration

    * schema updates

Dependencies
============

Dependencies to other developer teams have to be communicated to ensure they use
the old microversion to avoid breaking changes and to switch to the new split up
fields. This change should especially be communicated to the nova team which
checks regularly for the status of an attached volume.


Testing
=======

* Since only a reduced set of states are handled via the `status` field
  and with `backup_status` as newly introduced field, some functional tests
  have to be adapted to use the new data model.

* Further tests have to be added to ensure the translation layer for older API
  microversions work as expected. E.g. the backup status is presented via either
  `status` for an older microversion and then via `backup_status` for the
  new version.

* Because the `conditional_update`` method needs to support versioning in this
  release, test should be written to verify that the versioning happens
  correctly.


Documentation Impact
====================

* Add a release note explaining the motivation and effect of the change

* Document the state-machines for the volume itself and the backup and
  restore tasks.

* Document the translation layer for the older microversions and how the
  translation behaves.



References
==========

[1] Previously proposed spec to add a task status: https://review.opendev.org/c/openstack/cinder-specs/+/818551


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2023.02
     - Introduced

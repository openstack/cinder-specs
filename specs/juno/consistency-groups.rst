..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Consistency Groups
==========================================

https://blueprints.launchpad.net/cinder/+spec/consistency-groups

Today in Cinder, every operation happens at the volume level. Consistency
Groups (CGs) are needed for Data Protection (snapshots and backups) and
Disaster Recovery (remote replication). This blueprint will be introducing
Consistency Groups and discussing how it can be applied to snapshots.

Remote replication is covered in a different blueprint and will not be
discussed here.
https://blueprints.launchpad.net/cinder/+spec/volume-replication

Problem description
===================

Consistency Group support will be added for snapshots in phase 1 (Juno).

Future:

* After the Consistency Group is introduced and implemented for snapshots,
  it may be applied to backups. That will be after phase 1.

* Modify Consistency Group (adding existing volumes to CG and removing volumes
  from CG after it is created) will be supported after phase 1.

Assumptions:

* Cinder provides APIs that can be consumed by an orchestration layer.

* The orchestration layer has knowledge of what volumes should be grouped
  together.

* Volumes in a CG belong to the same backend.

* Volumes in a CG can have different volume types.


3 levels of quiesce:

* Application level: Not in Cinder's control

* Filesystem level: Cinder can call newly proposed Nova admin quiesce API
  which uses QEMU guest agent to freeze the guest filesystem before taking a
  snapshot of CG and thaw afterwards. However, this freeze feature in QEMU
  guest agent was just added to libvirt recently, so we can't rely on it yet.

* Storage level: Arrays can freeze IO before taking a snapshot of CG.  We can
  only rely on the storage level quiesce in phase 1 because the freeze feature
  mentioned above is not ready yet.

Proposed change
===============

Consistency Groups work flow

* Create a CG, specifying volume types that can be supported by this CG.

  * Scheduler will figure out which backend can handle these volume types.
    The backend (host) will be saved in the CG table in the db.

* Create a volume and specify the CG.

  * This step will be repeated until all volumes are created for the CG.

* Create a snapshot of the CG.

  * Cinder API creates cgsnapshot and individual snapshot entries in the db
    and sends request to Cinder volume node.

  * Cinder manager calls novaclient which calls a new Nova admin API "quiesce"
    that uses QEMU guest agent to freeze the guest filesystem. Can leverage
    this work:
    https://wiki.openstack.org/wiki/Cinder/QuiescedSnapshotWithQemuGuestAgent
    (Note: This step will be on hold for now because the freeze feature is not
    reliable yet.)

  * Cinder manager calls Cinder driver.

  * Cinder driver communicates with backend array to create a point-in-time
    consistency snapshot of the CG.

  * Cinder manager calls novaclient which calls a new Nova admin API
    "unquiesce" that uses QEMU guest agent to thaw the guest filesystem.
    (Note: This step will be on hold for now.)

Alternatives
------------

One alternative is to not implement CG in Cinder, but rather implementing it
at the orchestration layer.  However, in that case, Cinder wouldn't know which
volumes are belonging to a CG.  As a result, user can delete a volume belonging
to the CG using Cinder CLI or Horizon without knowing the consequences.

Another alternative is not to implement CG at all.  User will be able to
operate at individual volume level, but can't provide crash consistent data
protection of multiple volumes in the same application.

Data model impact
-----------------

DB Schema Changes

* A new consistencygroups table will be created.

* A new cgsnapshots table will be created.

* Volume entries in volumes tables will have a foreign key of the
  consistencygroup uuid that they belong to.

* cgsnapshot entries in cgsnapshots table will have a foreign key of the
  consistencygroup uuid.

* snapshot entries in snapshots table will have a foreign key of the
  cgsnapshot uuid.

::

 mysql> desc cgsnapshots;
 +---------------------+--------------+------+-----+---------+-------+
 | Field               | Type         | Null | Key | Default | Extra |
 +---------------------+--------------+------+-----+---------+-------+
 | created_at          | datetime     | YES  |     | NULL    |       |
 | updated_at          | datetime     | YES  |     | NULL    |       |
 | deleted_at          | datetime     | YES  |     | NULL    |       |
 | deleted             | tinyint(1)   | YES  |     | NULL    |       |
 | id                  | varchar(36)  | NO   | PRI | NULL    |       |
 | consistencygroup_id | varchar(36)  | YES  |     | NULL    |       |
 | user_id             | varchar(255) | YES  |     | NULL    |       |
 | project_id          | varchar(255) | YES  |     | NULL    |       |
 | name                | varchar(255) | YES  |     | NULL    |       |
 | description         | varchar(255) | YES  |     | NULL    |       |
 | status              | varchar(255) | YES  |     | NULL    |       |
 +---------------------+--------------+------+-----+---------+-------+
 11 rows in set (0.00 sec)

 mysql> desc consistencygroups;
 +-------------------+--------------+------+-----+---------+-------+
 | Field             | Type         | Null | Key | Default | Extra |
 +-------------------+--------------+------+-----+---------+-------+
 | created_at        | datetime     | YES  |     | NULL    |       |
 | updated_at        | datetime     | YES  |     | NULL    |       |
 | deleted_at        | datetime     | YES  |     | NULL    |       |
 | deleted           | tinyint(1)   | YES  |     | NULL    |       |
 | id                | varchar(36)  | NO   | PRI | NULL    |       |
 | user_id           | varchar(255) | YES  |     | NULL    |       |
 | project_id        | varchar(255) | YES  |     | NULL    |       |
 | host              | varchar(255) | YES  |     | NULL    |       |
 | availability_zone | varchar(255) | YES  |     | NULL    |       |
 | name              | varchar(255) | YES  |     | NULL    |       |
 | description       | varchar(255) | YES  |     | NULL    |       |
 | status            | varchar(255) | YES  |     | NULL    |       |
 +-------------------+--------------+------+-----+---------+-------+
 12 rows in set (0.00 sec)


Alternatives:

Instead of adding a cgsnapshots table, add a label to the snapshots.
This label will be the cgsnapshot name. That means we need to make sure
the name is provided when creating a snapshot of the CG and it must be unique.

REST API impact
---------------

Consistency Groups

Add V2 API extensions consistencygroups

* Create consistency group API

  * Create a consistency group.

  * Method type: POST

  * Normal Response Code: 202

  * Expected error http response code(s): TBD
    * 404: type group not found

  * V2/<tenant id>/consistencygroups

  * JSON schema definition for V2::

        {
            "consistencygroup":
            {
                "name": "my_cg",
                "description": "My consistency group",
                "volume_types": [type1, type2, ...],
                "availability_zone": "zone1:host1"
            }
        }


* Delete consistency group API

  * Delete a consistency group.

  * Method type: DELETE

  * Normal Response Code: 202

  * Expected error http response code(s):
    * 404: consistency group not found
    * 403: consistency group in use

  * V2/<tenant id>/consistencygroups/<cg uuid>

  * This API has no body.


* List consistency group API

  * This API lists summary information for all consistency groups.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s): TBD

  * V2/<tenant id>/consistencygroups

  * This API has no body.


* List consistency groups (detailed) API

  * This API lists detailed information for all consistency groups.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s): TBD

  * V2/<tenant id>/consistencygroups/detail

  * This API has no body.


* Show consistency group API

  * This API shows information about a specified consistency group.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s)
    * 404: consistency group not found

  * V2/<tenant id>/consistencygroups/<cg uuid>

  * This API has no body.

* Modify consistency group API (adding existing volumes to or removing
  volumes from the CG) will be addressed after phase 1.

* Create volume API will have "consistencygroup_id" added::

        {
            "volume":
            {
                ........
                ........
                "consistencygroup_id": "consistency group uuid",
                ........
                ........
            }
        }


Snapshots

Add V2 API extensions for snapshots of consistency group

* Create snapshot API

  * Create a consistency group.

  * Method type: POST

  * Normal Response Code: 202

  * Expected error http response code(s): TBD
    * 404: snapshot not found

  * V2/<tenant id>/consistencygroups/<cg uuid>/snapshots

  * JSON schema definition for V2::

        {
            "snapshot":
            {
                "name": "my_cg_snapshot"
                "description": "Snapshot of my consistency group"
            }
        }


* Delete snapshot API

  * Delete a snapshot of a consistency group.

  * Method type: DELETE

  * Normal Response Code: 202

  * Expected error http response code(s)
    * 404: snapshot not found

  * V2/<tenant id>/consistencygroups/<cg uuid>/snapshots/<snapshot id>

  * JSON schema definition for V2: None

  * Should not be able to delete individual volume snapshot if part of a
    consistency group.


* List snapshots API

  * This API lists summary information for all snapshots of a
    consistency group.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s): TBD

  * V2/<tenant id>/consistencygroups/<cg uuid>/snapshots

  * This API has no body.


* List consistency groups (detailed) API

  * This API lists detailed information for all snapshots of a
    consistency group.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s): TBD

  * V2/<tenant id>/consistencygroups/<cg uuid>/snapshots/detail

  * This API has no body.


* Show snapshot API

  * This API shows information about a specified snapshot of a
    consistency group.

  * Method type: GET

  * Normal Response Code: 200

  * Expected error http response code(s)
    * 404: snapshot of the consistency group not found

  * V2/<tenant id>/consistencygroups/<cg uuid>/snapshots/<snapshot id>

  * This API has no body.


Driver API additions

* def create_consistencygroup(self, context, consistencygroup, volumes)

* def delete_consistencygroup(self, context, consistencygroup)

* def create_cgsnapshot(self, context, cgsnapshot)

* def delete_cgsnapshot(self, context, cgsnapshot)

Security impact
---------------

None

Notifications impact
--------------------

Add event notifications.

Other end user impact
---------------------

Add a quota for maximum number of CGs per tenant.


python-cinderclient needs to be changed to support CG.  The following CLI
will be added.

To list all consistency groups:
 cinder consistencygroup-list

To create a consistency group:
 cinder consistencygroup-create --name <name> --description <description>
 --volume_type <type1,type2,...>

Example:
 cinder consistencygroup-create --name mycg --description "My CG"
 --volume_type lvm-1,lvm-2

To create a new volume and add it to the consistency group:
 cinder create --volume_type <type> --consistencygroup <cg uuid or name> <size>

To delete one or more consistency groups:
 cinder consistencygroup-delete <cg uuid or name> [<cg uuid or name> ...]

 cinder consistencygroup-show <cg uuid or name>


python-cinderclient needs to be changed to support snapshots.

To list snapshots of a consistency group:
 cinder consistencygroup-snapshot-list <cg uuid or name>

To create a snapshot of a consistency group:
 cinder consistencygroup-snapshot-create <cg uuid or name>

To show a snapshot of a consistency group:
 cinder consistencygroup-snapshot-show <cgsnapshot uuid or name>

To delete one or more snapshots:
 cinder consistencygroup-snapshot-delete <cgsnapshot uuid or name>
 [<cgsnapshot uuid or name> ...]


Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

This will add CG support to Cinder.  Other drivers can implement the proposed
driver APIs to support this feature.  This is not required.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xing-yang (xing.yang@emc.com)

Other contributors:
  None

Work Items
----------

* Add Cinder APIs.
* Make db schema changes.
* Driver API changes.
* Implement driver changes for LVM.
* Tempest tests.

Dependencies
============

None

Testing
=======

In order to test this feature, tests need to be added to Tempest to support
all new APIs.

Documentation Impact
====================

Need to document the new APIs.

References
==========

**Juno design session:**

https://etherpad.openstack.org/p/juno-cinder-cinder-consistency-groups

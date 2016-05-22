..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Group Snapshots
===============

https://blueprints.launchpad.net/cinder/+spec/generic-volume-group

This spec is dependent on the generic volume group spec [1]. The purpose of
proposing this spec is to migrate the existing consistency groups support
to use the generic volume group constructs and also provide a common API
for group snapshots that works for all storage backends.

Problem description
===================

In Juno, we introduced consistency groups (CG) support into cinder. The
existing CG support in cinder only supports CG snapshot.

The existing CG support includes the following tables only used for CG.
* consistencygroups
* cgsnapshot

The existing CG support provides the following APIs.
* Create CG
* Delete CG
* Update CG
* Create CG from source (CG or CG snapshot)
* List CG
* Show CG
* Create CG snapshot
* Create CG snapshot
* List CG snapshot
* Show CG snapshot

The generic volume group spec introduces a ``groups`` table which is
corresponding to the existing ``consistencygroups`` table.

The generic volume group spec introduces the following APIs that are
corresponding to some existing CG APIs.
* Create group
* Delete group
* Update group
* List group
* Show group

The missing pieces to support the existing CG feature using the generic
volume contructs are as follows.
* A group_snapshots table (for cgsnapshots table)
* Create group snapshot API (for create cgsnapshot)
* Delete group snapshot API (for delete cgsnapshot)
* List group snapshots API (for list cgsnapshots)
* Show group snapshot API (for show cgsnapshot)
* Create group from source group or source group snapshot API (for
  create CG from cgsnapshot or source CG)

Therefore we are proposing to add the missing pieces in this spec and
also provide a way to migrate data from the CG and CGSnapshot tables to
group and group snapshots tables while maintaining the support for
rolling upgrade.

The existing CG and CG snapshot APIs can only be supported by a subset
of storage backends in cinder. In this spec, we propose to provide group
snapshot APIs that can be supported by all storage backends.

Use Cases
=========

Group snapshot supports two capabilities, that is
consistent_group_snapshot_enabled and group_snapshot_enabled. A group snapshot
with consistent_group_snapshot_enabled spec set to True is equivalent to
cgsnapshot that is existing today in Cinder and it can gurantee point-in-time
consistency at the storage level. A group snapshot with group_snapshot_enabled
spec set to True is a group of snapshots that does not guarantee consistency
at the storage level.

Data protection products built on top of cinder, nova, neutron, glance, etc.
want to protect all OpenStack resources. They can use the group snapshot API
to take snapshots in their solution. Without the group snapshot API, the
data protection products will have to take snapshot for each volume
inidividually.

Proposed change
===============

* Create a group snapshot

  * Add an API that allows a tenant to create a group snapshot.
  * Add a Volume Driver API accordingly.

* Delete a group snapshot

  * Add an API that allows a tenant to delete a group snapshot.
  * Add a Volume Driver API accordingly.

* List group snapshots

  * Add an API to list group snapshots.

* Show a group snapshot

  * Add an API to show a group snapshot.

* Create a group from a source group or a source group snapshot

  * Add an API that creats a group from a source group or a
    source group snapshot.
  * Add a Volume Driver API accordingly.

* DB Schema Changes

  * A new group_snapshots table will be created and will contain the following:
    * uuid of the group_snapshot
    * name
    * description
    * uuid of the original group as a foreign key

  * A group_snapshot_id column will be added to the snapshots table.

  * Two new columns group_snapshot_id and source_group_id will be
    added to the groups table.

* Changes will be made to make sure generic volume group and group snapshots
  can support CG and CG snapshots.

  * Create a default group type and use it only for the existing CGs.

  * Write a migration script to copy data from consistencygroups to
    groups and from cgsnapshots to group_snapshots tables. All existing
    consistencygroups moved to groups will use the default group type.

  * In the future (i.e., Ocata) we can provide a cinder manage command
    to allow admin to change the group type.

  * In Newton, to support rolling upgrade, all existing CG and CG snapshot
    APIs will continue to work, and they will write to both existing and new
    tables. Read will be from the existing tables. So list and show will
    retrieve data from the existing tables.

  * In the Ocata release, using old CG APIs will still work and data will be
    written in both the new and old tables. Read will be from the new tables.
    So list and show will retrieve data from the new tables.

  * In the "P" release, using old CG APIs will still work and both write and
    read will be using the new tables. The old tables will be removed in the
    "P" release.

  * Using new APIs will write to and read from the new tables only. This means
    list groups will list all groups in the new tables including those created
    using CG APIs. The same applies to group snapshots. By checking the group
    type of a group, you can tell what kind of group it is.

  * When creating a group using the new API, if the following is in the group
    type spec, the manager will call create_group in the driver first and will
    call create_consistencygroup in the driver if create_group is not
    implemented.
        {'consistent_group_snapshot_enabled': <is> True}
    Same applies to delete_group, update_group, create_group_snapshot,
    delete_group_snapshot, and create_group_from_src. This way the new APIs
    will work with existing driver implementation of CG functions.

  * During the "P" release, we can make a decision on whether to keep the
    CG and CG snapshots APIs or deprecate them in the "Q" release.

Alternatives
------------

We can continue to use the existing CG and CG snapshot APIs.

Data model impact
-----------------

* DB Schema Changes

  * A new group_snapshots table will be created and will contain the following.

    * uuid of the group_snapshot
    * name
    * description
    * uuid of the original group as a foreign key

  * A group_snapshot_id column will be added to the snapshots table.

  * Two new columns group_snapshot_id and source_group_id will be
    added to the groups table.

REST API impact
---------------

New Group Snapshot APIs

* Create a Group Snapshot

  * V3/<tenant id>/group_snapshots
  * Method: POST
  * JSON schema definition for V3::

        {
            "group_snapshot":
            {
                "name": "my_group_snapshot",
                "description": "My group snapshot",
                "group_id": group_uuid,
                "user_id": user_id,
                "project_id": project_id,
            }
        }


* Delete Group Snapshot

  * V3/<tenant id>/group_snapshots/<group snapshot uuid>
  * Method: DELETE
  * This API has no body


* List Group Snapshots

  * V3/<tenant id>/group_snapshots
  * This API lists summary information for all group snapshots.
  * Method: GET
  * This API has no body.


* List Group Snapshots (detailed)

  * V3/<tenant id>/group_snapshots/detail
  * This API lists detailed information for all group snapshots.
  * Method: GET
  * This API has no body.


* Show Group Snapshot

  * V3/<tenant id>/group_snapshots/<group snapshot uuid>
  * Method: GET
  * This API has no body.


* Create Group from Source

 * V3/<tenant id>/groups/action
 * Method: POST
 * JSON schema definition for V3::

        {
            "create-from-src":
            {
                "name": "my_group",
                "description": "My group",
                "group_snapshot_id": group_snapshot_uuid,
                "source_group_id": source_group_uuid,
                "user_id": user_id,
                "project_id": project_id,
            }
        }


* Changes to Create Snapshot API

  * A new field "group_snapshot_id" (uuid of the group snapshot)  will be
    added to the request body.


* Cinder Volume Driver API

  The following new volume driver APIs will be added:

  * def create_group_snapshot(self, context, group_snapshot, snapshots)
  * def delete_group_snapshot(self, context, group_snapshot, snapshots)
  * def create_group_from_src(self, context, group, volumes,
                              group_snapshot=None, snapshots=None,
                              source_group=None, source_vols=None)


Security impact
---------------
None.

Notifications impact
--------------------
Notifications will be added for create and delete group snapshots and
create group from source.

Other end user impact
---------------------

python-cinderclient needs to be changed to support the new APIs.

* Create Group Snapshot

  cinder group-snapshot-create --name <name> --description <description>
  <group uuid>

* Delete Group Snapshot

  cinder group-snapshot-delete <group snapshot uuid>
  [<group snapshot uuid> ...]

* List Group Snapshot

  cinder group-snapshot-list

* Show Group Snapshot

  cinder group-snapshot-show <group snapshot uuid>

* Create Group from Source
  cinder group-create-from-src --group-snapshot <group snapshot uuid>
  --source-group <source group uuid> --name <name>
  --description <description>

Performance Impact
------------------
None

Other deployer impact
---------------------

None

Developer impact
----------------

Driver developers can implement the new driver APIs.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xing-yang

Other contributors:

Work Items
----------

1. New Group Snapshot APIs

   * Create Group Snapshot
   * Delete Group Snapshot
   * List Group Snapshots
   * Show Group Snapshot

2. New Clone Group API

   * Create Group from Source Snapshot or Source Group

3. New Volume Driver API changes

   * Create Group Snapshot
   * Delete Group Snapshot
   * Create Group from Source

4. New DB schema changes

5. Implement methods in the LVM driver.

6. Make sure both new and old APIs work. See details in the
   Proposed Change section.

Dependencies
============

Testing
=======

New unit tests will be added to test the changed code.
Tempest tests should be added as well.
Functional tests could be added if needed.

Documentation Impact
====================

Documentation changes are needed.

References
==========

[1] The generic volume group spec:

https://review.openstack.org/#/c/303893/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Generic Volume Group
====================

https://blueprints.launchpad.net/cinder/+spec/generic-volume-group

In this spec, we are designing a generic volume group. A generic volume group
can be used by a driver for various purposes and it can be extended easily
in the future to add new features.

Problem description
===================

In Cinder, the only group construct available today is Consistency Group.
Consistency Group only supports consistent group snapshot today. When we
started to look at extending this support for group replication, lots of
issues came up. Consistency Group implies data consistency. For some drivers,
a group replication can ensure data consistency across the volumes in the
group, but for other drivers, group replication does not imply data
consistency.

In this spec, we are designing a generic volume group. This group construct
can form a base for adding group replication support which will be discussed
in a different spec. The reason for adding a generic volume group is that it
can be used by drivers for different purposes. In this spec, we will be
providing basic functions including create, delete, and update a volume group.
Once the basic generic volume group is added, we can easily extend it to
support more new features without adding complex new APIs.

In the following section, use cases are added to explain why a group
construct is needed.

Use Cases
=========

* Group volumes together for a common purpose
  A tenant may want to put volumes used in the same application together in
  a group so that it is easier to manage them together.

* Group replication
  It was advised during previous spec reviews that generic volume groups
  should be split out from group replication and covered in separate specs.
  As suggested, group replication is discussed in details in a separate spec
  in [1].

  Once a volume group construct is available, it provides the base for adding
  support for group replication. Proposed replication functions in [1]
  only apply to a group that has group_replication_enabled or
  consistent_group_replication_enabled spec set to True. This means if you
  have created a group that does not have group_replication_enabled or
  consistent_group_replication_enabled spec set to True, you cannot
  enable replication on that group.

* Group snapshot
  Details on group snapshot are discussed in another spec [2]. Once a volume
  group construct is available, it provides the base for adding support for
  group snapshot. We also need to change the existing Consistency Group
  snapshot API to adopt the new group construct and make sure it does not
  break rolling upgrades.

  There is a difference between group snapshot propsed in [2] versus
  cgsnapshot which is existing today. Group snapshot supports two
  capabilities, that is consistent_group_snapshot_enabled and
  group_snapshot_enabled. A group snapshot with
  consistent_group_snapshot_enabled spec set to True is equivalent to
  cgsnapshot that is existing today. A group snapshot with
  group_snapshot_enabled spec set to True is a group of snapshots that does
  not guarantee consistency at the storage level. More details are provided
  in [2].

  Existing work flow:
  1. consisgroup-create creates a CG which is a group of volumes. Some
  drivers call array APIs to put volumes in a group on the array so that it
  can take a consistent snapshot later, but other drivers do not really have
  an array API at this step so it is just that Cinder puts the volumes in a
  group in the db. The next command cgsnapshot-create is the one that
  guarantees point-in-time consistency.
  2. cgsnapshot-create creates a CG snapshot which is consistent group
  snapshot.

  New work flow:
  1. group-create
  2. group-snapshot-create
  Whether the group snapshot is consistent or not depends on group type and
  capabilities reported by the driver.

Proposed change
===============

* Create a group

  * Add an API that allows a tenant to create a volume group.
  * Add a Volume Driver API accordingly.

* Delete a group

  * Add an API that allows a tenant to delete a volume group.
  * Add a Volume Driver API accordingly.

* List groups

  * Add an API to list volume groups.

* Show a group

  * Add an API to show a volume group.

* Update a group

  * Add an API that adds existing volumes to a group and removes volumes
    from the group after it is created.
  * Add a Volume Driver API accordingly.
  * Note: This is the ability to add an existing volume to a group or remove
    a volume from a group without deleting it. Some driver maintainers have
    reported that update group cannot be supported by their drivers. For
    example, HNAS iSCSI driver cannot support update group while implementing
    the existing CG features because it means moving files between filesystems
    and it does not have API to do that yet [3]. When implementing update
    group, the driver maintainer should evaluate backend capabilities and also
    check the group type specs to decide whether it can be supported or not.

* Group type

  * Add a group type for a group just like a volume type for a volume.
  * One group can support only one group type.

* Group type specs

  * Add group type specs to describe a group's characteristics just like
    volume type extra specs to a volume type.
  * Group type specs will be reported as "capabilities" similar to extra
    specs of a volume type.

* Changes in the scheduler.

  * Make changes in the scheduler so that an appropriate backend will be
    chosen to create a group.

* DB Schema Changes

  * A new group_type table will be created and will contain the following:
    * uuid of the group_type
    * name
    * description

  * A new group_type_specs table will be created.

    * uuid of the spec
    * key
    * value
    * uuid of group_type as a foreign key

  * A new group table will be created and will contain the following:

    * uuid of the group
    * uuid of a group_type as a foreign key
    * name
    * description

  * A new volume_group_mapping table will be created and will contain the
    following columns. This is needed because one volume could be in
    multiple groups.

    * uuid of the mapping entry
    * uuid of a volume
    * uuid of a group
    * Note: Restricting a volume to only one group will limit what we
      can do with the generic group contruct in the future.

  * A new group_volumetypes_mapping table will be created and will contain
    3 columns:

    * uuid of a group_volumetype entry
    * uuid of a group
    * uuid of a volume type

  * A group quota mechanism will be added, similar to what we have for
    CG.

* Changes will be made to make sure new group contruct can support CG.

  * In the Newton release, we should be able to create a CG using the
    new API and preserve the existing behavior of the CG API.
  * In the Ocata release, a group will be created in both the new and old
    tables and we will read from the old table.
  * In the "P" release, we will write to both new and old tables and read
    from the new table.
  * In the "Q" release, the old table will be removed. Both write and read
    will be from the new table.

* Driver needs to report group capabilities. Examples are as follows:

  * consistent_group_replication_enabled
  * group_replication_enabled
  * consistent_group_snapshot_enabled
  * group_snapshot_enabled

  Group type spec needs to specify capabilities, i.e.,
  {'group_snapshot_enabled': <is> True}

Alternatives
------------

Without these proposed changes, we can add replication support to the existing
consistency group but won't have a generic volume group that can serve more
purposes.

Data model impact
-----------------

* DB Schema Changes

  * A new group_type table will be created and will contain the following:

    * uuid of the group_type
    * name
    * description

  * A new group table will be created and will contain the following:

    * uuid of the group
    * uuid of a group_type as a foreign key
    * name
    * description

  * A new group_specs table will be created and will contain the following:

    * uuid of the group_spec
    * key
    * value
    * uuid of the group_type as a foreign key

  * A new group_volumetypes table will be created and will contain 3 columns:

    * uuid of a group_volumetype entry
    * uuid of a group
    * uuid of a volume type

  * The volumes table will have a new column:

    * uuid of the group as a foreign key

REST API impact
---------------

New Group Type APIs

* Create Group Type

  * V3/<tenant id of admin>/group_types
  * Method: POST
  * JSON schema definition for V3::

        {
            "group_type":
            {
                "name": "my_group_type",
                "description": "My group type",
                "group_type_specs": {"key1": "value1", "key2": "value2", ...}
            }
        }


* Delete Group Type

  * V3/<tenant id of admin>/group_types/<group_type_uuid>
  * Method: DELETE
  * This API has no body.


New Group Type Specs APIs

* Create Group Type Spec

  * V3/<tenant id of admin>/group_types/<group_type_uuid>/group_type_specs
  * Method: POST
  * JSON schema definition for V3::

        {
            "group_type_specs":
            {
                "key": "value"
            }
        }

* Delete Group Type Spec

  * V3/<tenant id of admin>/group_types/<group_type_uuid>/group_type_specs/
    <spec_uuid>
  * Method: DELETE
  * This API has no body.

* List Group Type Specs

  * V3/<tenant id of admin>/group_types/<group_type_uuid>/group_type_specs
  * Method: GET
  * This API has no body.


* Show Group Type Spec

  * V3/<tenant id of admin>/group_types/<group_type_uuid>/group_type_specs/
    <spec_uuid>
  * Method: GET
  * This API has no body.


New Group APIs

* Create a Group

  * V3/<tenant id>/groups
  * Method: POST
  * JSON schema definition for V3::

        {
            "group":
            {
                "name": "my_group",
                "description": "My group",
                "group_type": group_type_uuid,
                "volume_types": [volume_type1_uuid, volume_type2_uuid, ...],
                "availability_zone": "az1"
            }
        }

  * Cinder scheduler will find a backend that supports the group type and
    volume types. One group will be hosted on one backend, similar to CG.
    An empty group will be created first with the above API. After the
    group is created, a tenant can create a volume providing the group
    uuid and volume type and the volume will be provisioned and placed
    in the group and on the backend where the group resides.

  * The reason to include volume types when creating a group is to make sure
    that the backend selected to place the group will be able to host a volume
    of the specified volume type at a later time.

  * One group can support one group type and multiple volume types.

  * Cinder API will be responsible for creating the group entry in the
    database. Cinder driver may or may not need to do anything special when
    the empty group is first created. This depends on the backend. Most
    drivers just need to return success.


* Update Group

  * V3/<tenant id>/groups/<group uuid>
  * Method: PUT
  * JSON schema definition for V3::

        {
            "group":
            {
                "name": "my_group",
                "description": "My group",
                "add_volumes": [volume uuid 1, volume uuid 2,...]
                "remove_volumes": [volume uuid 8, volume uuid 9,...]
            }
        }

  * This method can update name, description, as well as volumes in the group.
    The list after "add_volumes" will contain UUIDs of volumes to be added to
    the group and the list after "remove_volumes" will contain UUIDs of volumes
    to be removed from the group. The API will validate the input name,
    description, UUIDs in add_volumes and remove_volumes fields against the
    information in Cinder db and send the request to the volume manager.
    Manager will call driver to do the update on the backend. The API will
    update Cinder db.


* Delete Group

  * V3/<tenant id>/groups/<group uuid>/action
  * Method: POST (We need to use "POST" not "DELETE" here because the request
    body has a flag and therefore is not empty.)
  * JSON schema definition for V3::
        {
            "delete":
            {
                "delete-volumes": False
            }
        }
  * Set delete-volumes flag to True to delete a group with volumes in it.
    This will delete the group and all the volumes. Deleting an empty
    group does not need the flag to be True.

* List Groups

  * V3/<tenant id>/groups
  * This API lists summary information for all groups.
  * Method: GET
  * This API has no body.


* List Groups (detailed)

  * V3/<tenant id>/groups/detail
  * This API lists detailed information for all groups.
  * Method: GET
  * This API has no body.


* Show Group

  * V3/<tenant id>/groups/<group uuid>
  * Method: GET
  * This API has no body.


* Changes to Create Volume API

  * A new field "group_id" (uuid of the group)  will be added to the
    request body.


* Cinder Volume Driver API

  The following new volume driver APIs will be added:

  * def create_group(self, context, group)
  * def update_group(self, context, group, add_volumes=None,
    remove_volumes=None)
  * def delete_group(self, context, group, volumes)


Security impact
---------------
None.

Notifications impact
--------------------
Notifications will be added for create, delete, and update groups.

Other end user impact
---------------------

python-cinderclient needs to be changed to support the new APIs.

* Create Group Type

  cinder group-type-create --description <description> <name>

* Delete Group Type

  cinder group-type-delete <group type uuid>

* Show Group Type

  cinder group-type-show <group type uuid>

* List Group Types

  cinder group-type-list

* Set/unset Group Type Spec

  cinder group-type-key <group type name or uuid> <action> <key-value>
  Valid action is "set" or "unset".

* List Group Type Specs

  cinder group-type-specs-list

* Create Group

  cinder group-create --name <name> --description <description>
  --availability-zone <availability-zone>
  --volume-types <volume type uuid> [<volume type uuid>...] group_type

* Update Group

  cinder group-update --name <new name>
  --description <new description> --add-volumes
  <volume uuid> [<volume uuid> ...] --remove-volumes
  <volume uuid> [<volume uuid> ...] <group uuid or name>

* Delete Group

  cinder group-delete --delete-volumes <group uuid> [<group uuid> ...]
  The ``delete-volumes`` flag is needed when a group is not empty.

* List Group

  cinder group-list

* Show Group

  cinder group-show <group uuid>

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

1. New Group Type APIs:

   * Create Group Type
   * Delete Group Type

2. New Group Type Spec APIs:

   * Create Group Type Spec
   * Delete Group Type Spec
   * List Group Type Specs
   * Show Group Type Spec

3. New Group APIs:

   * Create Group
   * Update Group
   * Delete Group
   * List Groups
   * Show Group

4. New Volume Driver API changes:

   * Create Group
   * Update Group
   * Delete Group

5. DB schema changes

6. Implement group methods in the LVM driver.

7. Make sure both new and old group APIs work.
   See details in spec [2] on how to achieve this.

Dependencies
============

Testing
=======

New unit tests will be added to test the changed code.

Documentation Impact
====================

Documentation changes are needed.

References
==========

[1] The replication group spec:
    https://review.openstack.org/#/c/229722/

[2] The group snapshot spec:
    https://review.openstack.org/#/c/331397/

[3] HNAS iSCSI driver Consistency Group support code review:
    https://review.openstack.org/#/c/327043/4/cinder/volume/drivers/
    hitachi/hnas_iscsi.py@939

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Consistency Groups Kilo Update
==============================

https://blueprints.launchpad.net/cinder/+spec/consistency-groups-kilo-update

Consistency Groups support was introduced in Juno. This proposal is to
enhance it by adding a few new features.

Problem description
===================

* Create CG from CG snapshot

  Currently a user can create a Consistency Group and create a snapshot of a
  Consistency Group.  To restore from a Cgsnapshot, however, the following
  steps need to be performed:

  1) Create a new Consistency Group.
  2) Do the following for every volume in the Consistency group:

     a) Call "create volume from snapshot" for snapshot associated with every
        volume in the original Consistency Group.

  There's no single API that allows a user to create a Consistency Group from
  a Cgsnapshot.

* Modify CG
  After a CG is created and volumes are created and added to CG, you can
  delete the entire CG with all volumes in it, but there's no API to add an
  existing volume to the CG or remove a volume from it.

* DB Schema Changes
  Volume types are required when creating a Consistency Group.  Currently
  volume types are stored in a field in the consistencygroups table in the
  Cinder database.  There's a limitation to this approach, however, because
  the size of the column is fixed.

Use Cases
=========

Proposed change
===============

* Create CG from CG snapshot

  * Add an API that allows a user to create a Consistency Group from a
    Cgsnapshot.

  * Add a Volume Driver API accordingly.

* Modify Consistency Group

  * Add an API that adds existing volumes to CG and removing volumes from CG
    after it is created.

  * Add a Volume Driver API accordingly.

* DB Schema Changes

  The following changes are proposed:

  * A new cg_volumetypes table will be created.
  * This new table will contain 3 columns:

    * uuid of a cg_volumetype entry
    * uuid of a consistencygroup
    * uuid of a volume type

  * Upgrade and downgrade functions will be provided for db migrations.

Alternatives
------------

Without these propsed changes, we have to deal with the current limitations.

Data model impact
-----------------

* DB Schema Changes
  The following changes are proposed:
  * A new cg_volumetypes table will be created.
  * This new table will contain 3 columns:

    - uuid of a cg_volumetype entry
    - uuid of a consistencygroup
    - uuid of a volume type

REST API impact
---------------

New Consistency Group APIs changes

* Create Consistency Group from Cgsnapshot
  * V2/<tenant id>/consistencygroups
  * Method: POST
  * JSON schema definition for V2::

        {
            "consistencygroup":
            {
                "name": "my_cg",
                "description": "My consistency group",
                "cgsnapshot": my_cgsnapshot,
            }
        }

  * In the Create Consistency Group API, if cgsnapshot is not specified, the
    code path stays the same as before and the request goes to the scheduler;
    If cgsnapshot is specified, the request will be sent to the backend where
    the original Consistency Group resides.

  * Cinder API will be responsible for creating the consistencygroup entry
    and volume entries in the database. Cinder driver will be responsible for
    creating it in the backend.

* Update Consistency Group
  * V2/<tenant id>/consistencygroups/<cg uuid>
  * Method: PUT
  * JSON schema definition for V2::

        {
            "consistencygroup":
            {
                "name": "my_cg",
                "description": "My consistency group",
                "addvolumes": [volume uuid 1, volume uuid 2,...]
                "removevolumes": [volume uuid 8, volume uuid 9,...]
            }
        }

  * This method can update name, description, as well as volumes in the
    consistency group. The list after "addvolumes" will contain UUIDs of
    volumes to be added to the group and the list after "removevolumes"
    will contain UUIDs of volumes to be removed from the group. The API
    will validate the input name, description, UUIDs in addvolumes and
    removevolumes fields against the information in Cinder db and send
    the request to the volume manager. Manager will call driver to do
    the update on the backend. The API will update Cinder db.


* Cinder Volume Driver API

  The following new volume driver APIs will be added:

  .. code-block:: python

    def create_consistencygroup_from_cgsnapshot(self, context,
        consistencygroup, volumes, cgsnapshot, snapshots)

    def modify_consistencygroup(self, context, consistencygroup,
        old_volumes, new_volumes)

Security impact
---------------


Notifications impact
--------------------


Other end user impact
---------------------

python-cinderclient needs to be changed to support the new APIs.

* Create CG from CG snapshot

  .. code-block:: bash

    cinder consisgroup-create --name <name> --description <description>
    --cgsnapshot <cgsnapshot uuid or name>

* Modify CG

  .. code-block:: bash

    cinder consisgroup-modify <cg uuid or name> --name <new name>
    --description <new description> --addvolumes
    <volume uuid> [<volume uuid> ...] --removevolumes
    <volume uuid> [<volume uuid> ...]

Performance Impact
------------------


Other deployer impact
---------------------

None. The db schema changes are internal and should be transparent to
end users.

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

1. API changes:

   * Create CG from CG snapshot API
   * Modify CG API

2. Volume Driver API changes:

   * Create CG from CG snapshot
   * Modify CG

3. DB schema changes

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


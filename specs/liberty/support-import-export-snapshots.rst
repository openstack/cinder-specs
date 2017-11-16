..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Support import/export snapshots in cinder
=========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/support-import-export-snapshots

Provide a mean to import/export snapshots. Import/export snapshots function
could be an complement for the function of import/export volume.

Problem description
===================

Import/export snapshots function:

* It could provide the ability to import volumes' snapshot from one cinder
  to another cinder, and and import "non" OpenStack snapshots already on a
  back-end device into OpenStack/cinder.

* Export snapshots works the same way as export volumes.

Benefits:

* We could use the import snapshots as volume templates to create volumes from
  snapshots.

* Import snapshots function could provide an effective means to manage the
  import volumes.

  ::
    For those volume drivers which could not delete volume with snapshots,
    we could not delete the import volume that has snapshots.
    By using import snapshots function to import snapshots, we could first
    delete the import snapshots, and then delete the import volumes.

Use Cases
=========

Proposed change
===============

By supporting import/export snapshots, we need to do the following changes.

* Add import/export snapshots API.

* Add import/export snapshots work flow.

    ::
        Import snapshots would check whether the snapshot's parent volume
        exists in cinder. If not, it would raise an exception.NotFound.
        If the snapshot doesn't exist in volume back end or the volume doesn't
        have related snapshot, it would raise
        exception.ManageExistingInvalidReference.
        Currently import snapshots function doesn't support import those
        snapshots(generally created by storage's snap clone operation)
        without parent volumes. We should use import volumes function to
        import snapshots without parent volume.

        Export snapshots work almost the same as delete snapshot,
        but it doesn't delete the snapshot data.
        For those volume drivers which could directly force delete volumes
        with snapshots, export snapshots function has no side effect.
        For those volume drivers which could not delete volumes with
        snapshots, export snapshots function has a side effect that would
        cause delete volumes failed.
        For lvm driver and rbd driver, if unmange volume's snapshots and
        delete the volume, the delete operation fail and the volume's status
        is available because these two drivers would not continue to delete
        volume if the volume has snapshots. Drivers raise an VolumeIsBusy
        exception to volume manager, and volume manager reset the volume's
        status to available.

* Add manage_existing_snapshot, manage_existing_snapshot_get_size and
  unmanage_snapshot interface in volume driver and implement these two
  interface in LVM driver.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

1. Add one API "manage_snapshot"

The rest API look like this in v2:

.. code-block: console

  POST /v2/{project_id}/os-snapshot-manage

.. code-block:: python

    {
        'snapshot':{
        'host': 'cinder-volume',
        'volume_id': 'volume-058660ab-6771-4f82-9627-687b179175d6'
        'ref': '_snapshot-058660ab-6771-4f82-9627-687b179175d6'
        }
    }

  * The <string> 'host' means cinder volume host name.
    No default value.

  * The <string> 'volume_id' means the import snapshot's volume id.
    No default value.

  * The <string> 'ref' means the import snapshot name
    exist in storage back-end.
    No default value.

The response body of it is like:

.. code-block:: python

    {
        "snapshot": {
            "status": "creating",
            "description": null,
            "created_at": "2014-08-25T11:14:31.469591",
            "metadata": {},
            "volume_id": "fbd83a45-cce7-4333-b991-dafd2251edd4",
            "size": 1,
            "id": "71543ced-a8af-45b6-a5c4-a46282108a90",
            "name": null
        }
    }

  * The <string> 'status' will be creating.
  * The <string> 'description' means the import snapshot display name.
  * The <date> 'created_at' means the import time.
  * The <map> 'metadata' means the snapshot's metadata.
  * The <string> 'volume_id' means the snapshot's volume id.
  * The <string> 'id' means the snapshot's id.
  * The <string> 'name' means the alias of snapshot id.


2. Add one API "unmanage_snapshot".

The rest API look like this in v2:

.. code-block:: console

  POST /v2/{project_id}/os-snapshot-manage/{id}/action

.. code-block:: python

  {
      'os-unmanage':{}
  }

The status code will be HTTP 202 when the request has succeeded.

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

1. Users can import snapshots that are already exist in volume back-end.
2. Export snapshots function has an side effect for those volume drivers
   which could not delete volumes with snapshots. If using export snapshots
   function, it would cause the subsequent delete volume operation fail.

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
  ling-yun<zengyunling@huawei.com>


Work Items
----------

* Implement code that mentioned in "Proposed change".
* Implement code in python-cinderclient.
* Add change API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change" and ensure that Cinder snapshot feature works
well while introducing import/export snapshots.


Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.
2. Add the side effect of export snapshots function.

References
==========

None

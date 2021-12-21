..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Add volume re-image API
=======================

https://blueprints.launchpad.net/cinder/+spec/add-volume-re-image-api

This blueprint proposes to add volume re-image API to enable the ability to
re-image a specific volume.

Problem description
===================

Currently, users can create a volume with a specific image, but once this
volume is created, the image of this volume can't be changed anymore. If users
want to re-image a specific volume, they can only create a new volume with new
image, and then delete the old volume, which is not very convenient. However,
if the image-backed volume is already used as a boot volume which is already
attached to a server, it makes the case more complicated, because changing the
image in an attached root volume is `not supported`_.

In addition, as mentioned in nova spec 'volume-backend-server-rebuild' [1]_,
if users want to rebuild a volume-backed server, they can complete it by using
re-image API, this way there is no new volume with new volume id, we don't
have to worry about types, quota problem, etc.

As the discussion result during Yoga PTG [2]_, the idea still makes sense and
we would propose to implement a new API to support this.

.. _not supported: https://review.openstack.org/#/c/520660/

Use Cases
=========

* The user wants to re-image a volume without re-creating a new volume with new
  volume id.
* Nova wants a new API to re-image an attached root volume when the server is
  rebuilt.

Proposed change
===============

This spec proposes to add a new 'os-reimage' action to volume actions API.

* Add a new 'os-reimage' action to volume actions API.
* The request will be rejected if cinder microversion is not new enough to
  support 'os-reimage' action.

As mentioned above, the main purpose of this spec is for Nova's rebuild use
case. When Nova calls cinder for re-image then cinder will perform these following
steps:

#. See nova side operations in the nova spec here
   'volume-backend-server-rebuild' [1]_
#. Receive the re-image request from nova and check if the volume is in
   ``reserved`` state.
#. Perform the re-image operation. During this, the volume will transition
   into ``downloading`` state which is similar to the state when creating a
   bootable volume from image (more detailed information is captured later in
   this section).
#. Add a new 'volume-reimaged' external event and use it notify nova when the
   re-image operation is complete, like we use for volume-extend.
   See `perform_resize_volume_online`_ for details.
#. After successful completion of the re-image operation, nova will call cinder
   to update the attachment with connector info.
#. Cinder will update the attachment and return connection info to Nova.
#. After Nova completes the connection with brick, Nova will make the attachment
   complete call marking the volume ``in-use``.

.. _perform_resize_volume_online: https://review.opendev.org/c/openstack/nova/+/454322

So, we propose to add a new microversion to support users to re-image a
specific volume with new image id. Only ``reserved``, ``available`` and
``error`` volume can be re-imaged. The ``reserved`` volume can only be
re-imaged when the ``reimage_reserved`` parameter is ``True``. The volume
state will be changed to ``downloading`` first.

Two separate policies will be introduced, one for re-imaging an ``available``
or ``error`` volume and another for re-imaging an ``reserved`` volume, the
default policies for these new actions are ``SYSTEM_ADMIN_OR_PROJECT_MEMBER``.

If the image status is not ``active``, image size or image min_disk size is
larger than volume size, will raise InvalidInput (400) error.

* Add a new ``reimage`` cast rpc. This rpc is used to cast an async message
  from cinder API to the cinder-volume service or cluster.

* Add a new method for volume manager. There will be a new method for volume
  manager which will be called 'reimage', this method will first download the
  image from glance and then perform the re-image operation.

* Add an external event ``volume-reimaged`` and call it to notify nova API
  about completion of reimage operation.

We propose to only support the basic implementation in this spec, the
re-image method will call driver's 'copy_image_to_volume' to re-write the
specific volume with new image. If the specific image is encrypted, then it
will call driver's 'copy_image_to_encrypted_volume'. It's similar to what we
have done when we create volume from image.

After the re-image operation is complete, the volume will set back to
original state, and all previously existing volume_glance_metadata of this
volume will be replaced with the metadata from new image.

* If the re-image operation fails, we will set a user message to give hints to
  the user how they can recover from a potentially now corrupt boot volume and
  the volume status will changed to ``error``.

There are also some other optimized mechanisms that should be considered to
support in future, but will not be implemented in this spec:

* Re-image an ``in-use`` volume. If we want to support it, we should make
  sure this volume can be multi-attached, so that we can attach this volume to
  a host to support copy image to volume. And also re-image an ``in-use``
  volume is a very dangerous operation, we must make sure there is no IO,
  onging or cached on VMs before we complete re-imaging, otherwise will damage
  the volume data. We will not support this operation in this spec.

* Re-image a volume from image cache. Currently, Cinder supports creating
  a volume from image cache, which can improve the performance of
  creating a volume from an image, we can also consider to add this
  improvement in the advanced implementation.

* Add image signature verification support when re-imaging a volume. There is
  also a signature when we create volume from image, we will also support it
  in the advanced implementation. This will be called out in the API reference
  documentation as a limitation until it is implemented.

Alternatives
------------

If the users want to re-image a volume to same image, the users could revert
the volume to a snapshot created after image.

But for different image, the users could first delete the volume and then
re-create a volume with a specific new image. And nova could orchestrate
creating a new root volume with the new image, then replace it during the
server rebuild operation, but there are several issue with that as mentioned
above like quota and the questions about what nova should do about the old
volume, you can see more detail information in ML [3]_.

REST API impact
---------------

* A new microversion will be created to support volume 'os-reimage' action API

The rest API look like this in v3:

.. code-block:: console

  POST /v3/{project_id}/volumes/{volume_id}/action

.. code-block:: json

    {
        "os-reimage": {
            "image_id": "71543ced-a8af-45b6-a5c4-a46282108a90",
            "reimage_reserved": true
        }
    }

* The <string> 'image_id' refers to the id of image.
  No default value since this is a required parameter.
* The <boolean> 'reimage_reserved' refers to re-image a volume and ignore its
  'reserved' status. The 'available' and 'error' volume can be re-imaged
  directly, but the 'reserved' volume can only be re-imaged when this
  parameter is 'true'.
  Defaults to 'false', this is an optional parameter.

The response body of it is like:

.. code-block:: json

    {
      "volume": {
          "migration_status": null,
          "attachments": [],
          "links": [
              {
                  "href": "http://10.79.144.144/volume/v3/ffc60994a7274553905e5e5a8f890ab3/volumes/d90bfc0e-babf-4478-a591-23ca883ba2be",
                  "rel": "self"
              },
              {
                  "href": "http://10.79.144.144/volume/ffc60994a7274553905e5e5a8f890ab3/volumes/d90bfc0e-babf-4478-a591-23ca883ba2be",
                  "rel": "bookmark"
              }
          ],
          "availability_zone": "nova",
          "os-vol-host-attr:host": "ubuntubase@lvmdriver-1#lvmdriver-1",
          "encrypted": false,
          "updated_at": "2018-09-26T01:55:41.084080",
          "replication_status": null,
          "snapshot_id": null,
          "id": "d90bfc0e-babf-4478-a591-23ca883ba2be",
          "size": 2,
          "user_id": "f33299af48b44050b96bc51104f2290a",
          "os-vol-tenant-attr:tenant_id": "ffc60994a7274553905e5e5a8f890ab3",
          "os-vol-mig-status-attr:migstat": null,
          "metadata": { },
          "status": "downloading",
          "volume_image_metadata": {
              "container_format": "bare",
              "min_ram": "0",
              "disk_format": "qcow2",
              "image_name": "cirros-0.3.5-x86_64-disk",
              "image_id": "24d0c0c3-9e03-498c-926f-ac964cbe2e08",
              "checksum": "f8ab98ff5e73ebab884d80c9dc9c7290",
              "min_disk": "0",
              "size": "13267968"
          },
          "description": null,
          "multiattach": false,
          "source_volid": null,
          "consistencygroup_id": null,
          "os-vol-mig-status-attr:name_id": null,
          "name": null,
          "bootable": "true",
          "created_at": "2018-09-26T01:55:38.735749",
          "volume_type": "lvmdriver-1"
        }
    }

* The <string> 'status' will be 'downloading'.
* The <dict> 'volume_image_metadata' refers to the image metadata of the
  volume. It will include the original image until the re-image operation is
  complete in cinder-volume.

- Normal response codes: 202

- Error response codes: 400, 403, 404, 409

Data model impact
-----------------

None

Security impact
---------------

Two new policy rules will be introduced as noted in the `Proposed change`_
section.

Notifications impact
--------------------

Notifications will be added for re-image volume.

Other end user impact
---------------------

A new command, ``cinder reimage <volume_id> <image_id> [--ignore-reserved]``,
will be added to python-cinderclient. This command mirrors the underlying API
function.

Callers of the new API will need to poll the status of the volume until it
goes back to its original status or ``error`` in case the operation failed.
The only exception here is Nova which will be notified via external events
API and doesn't need to poll for the re-image to be completed.

Since this is a data path change, it will only modify the contents of volume and
dependent resources like snapshots or backups won't be affected by it. Just
to keep in mind that restoring to an earlier backup/snapshot will also revert
the volume to the old image.

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

Work Items
----------
By supporting re-image volumes, we need to do the following changes:

* Add a new 'os-reimage' action to volume actions API.

* Add a new 'reimage' cast rpc.

* Add a new 'reimage' method in volume manager.

* Adopt the new microversion in python-cinderclient.

* Set a user message in the event when the re-image fails.

* Add a call to nova external events API with 'volume-reimaged' event

Dependencies
============

None

Testing
=======

Unit-tests, tempest and other related tests will be implemented.

Documentation Impact
====================

Need to document the new behavior of the volume re-image, as well
as related client examples, etc.

We also need to mention in the documentation that when the volume
is re-imaged, all current content on the volume will be *destroyed*.
This is important as cinder volumes are considered to be persistent,
which is not the case with this operation.

References
==========

.. [1] https://review.opendev.org/c/openstack/nova-specs/+/809621

.. [2] https://wiki.openstack.org/wiki/CinderYogaPTGSummary#Volume_re-image

.. [3] http://lists.openstack.org/pipermail/openstack-operators/2018-March/014952.html

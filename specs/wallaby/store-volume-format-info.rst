..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Store volume format info in Cinder
==================================

https://blueprints.launchpad.net/cinder/+spec/add-support-store-volume-format-info


Problem description
===================

Currently, there are several filesystem-based drivers in Cinder which rely
on auto detection of formats. Cinder does not store the actual format of
volume and supposes all volumes are "raw" format.

One of such problems is seen when using glance cinder store with nfs as
cinder backend and the file to be copied is a qcow2 image. When performing
the extend operation, qemu-img uses the resize command which detects
the volume format as qcow2 by reading the image's qcow2 header written into
the volume. This leads to unsuccessful creation of the image.
This is a problem because current code depends on auto detection of format
and if the user has written a qcow2 image into the volume, the driver thinks
it is a qcow2 volume (rather than qcow2 image stored in a raw volume) hence
causing complication.
Thus we need to store format info instead of auto detection.


Proposed change
===============

The "format" info will be added to "volume_admin_metadata" of volumes.
The "format" will be only set / updated for filesystem-based drivers, other
drivers will not contain this metadata.
The "format" field will be created by FS drivers at the time of volume creation
and is set to the format which the driver is configured to use.
The "format" info will be returned in the attachment get API in the
connection_info dict.
When cloning a volume, this field will be updated in the new volume's metadata.
For existing volumes, we check if format exists in admin_metadata and if not,
we can perform ``qemu-img info`` to get the format and store it in the
admin_metadata at that point.

Alternatives
------------

Some of the fs type driver uses "qemu-img info" command to judge
the format of volume. However, this auto-detection method has many possible
error / exploit vectors. Because if the beginning content of a "raw" volume
happens to be a "qcow2" disk, auto detection method will judge this volume to be
a "qcow2" volume wrongly. This is the same issue which cases glance to fail
an image create operation when using cinder as a store.

Data model impact
-----------------

For volumes on filesystem-based drivers, there will be a related record in
volume_admin_metadata table. For this record, the key column is "format" and
the value is the volume's format in volume_admin_metadata table.

REST API impact
---------------

The "format" info will returned in attachment get API::

    /v3/{project_id}/attachments/{attachment_id}

    {
        "attachment": {
            "id": "3b8b6631-1cf7-4fd7-9afb-c01e541a073c",
            ...
            "connection_info": {
                "format": "raw",
                ...
            }
        }
    }

Security impact
---------------

Implements the proper/complete fix for bug 1350504.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

Fixes problem where a user can cause a volume to become inoperable
by writing a qcow2 header into it.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  whoami-rajat

Work Items
----------

To support this feature, the work items are:

* Store format info in ``volume_admin_metadata`` table when creating or
  cloning a volume in fs type drivers.
* Modify volume format when doing a snapshot delete (blockRebase) operation
  on an attached volume.
* Pass format info when doing an extend operation.
* Return format field in the attachment get API in connection_info dict.

Dependencies
============

None

Testing
=======

Related unit and functional tests.

Documentation Impact
====================

Add documentation regarding the format field visible in the attachment get API
for fs drivers.

References
==========

None

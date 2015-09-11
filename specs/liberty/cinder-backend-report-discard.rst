..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================
Add ability for Cinder backend to report discard/unmap/trim
===========================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-backend-report-discard

Currently, libvirt/qemu has support for a discard option when attaching a
volume to an instance. With this feature, the unmap/trim command can be sent
from guest to the physical storage device.

A cinder back-end should report a connection capability that can be used to
instruct Nova how to attach a volume.

This spec aims to add the assist to support enabling discard for cinder
volumes.

Problem description
===================

Currently there is no way for Nova to know if a Cinder back end supports
discard/trim/unmap functionality.  This aims to provide that info to Nova.

Use Cases
=========

If a Cinder backed uses media that can make use of discard functionality
there should be a way to do this.  This will improve long term performance
of such back ends.

Proposed change
===============

Any driver which wants to support this will just add an entry to the
properties returned by intialize_connection.

"discard": True,

An example from the Pure Storage driver would be

.. code:: Python

        properties = {
            "driver_volume_type": "iscsi",
            "data": {
                "target_iqn": target_port["iqn"],
                "target_portal": target_port["portal"],
                "target_lun": connection["lun"],
                "target_discovered": True,
                "access_mode": "rw",
                "discard": True,
            },
        }

Alternatives
------------

Alternatives include adding a setting to cinder.conf for each backend that
supports this. This includes a manual step that would be more error prone.

Data model impact
-----------------

None

REST API impact
---------------

Documentation on the response to 'initialize_connection' will need to be
updated with notes explaining that this new info is provided.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

There will be a performance gain for back ends that benefit from having
discard functionality.

See https://en.wikipedia.org/wiki/Trim_(computing) for more info.

Other deployer impact
---------------------

Deployers will need to add some properties to Glance images to make
this work.  Once an instance is booted from the modified image the
discard capability reported by the driver will be activated.

hw_scsi_model=virtio-scsi
hw_disk_bus=scsi

glance image-update --property hw_scsi_model=virtio-scsi
                    --property hw_disk_bus=scsi <image-uuid>

This causes the correct controller to be created on the instance
via code changes here https://review.openstack.org/#/c/70263/.
Once the correct controller is in place the unmap functionality
specified by the Cinder backend will be utilized.

A secondary volume attachment will be treated separately and will
honor the discard settings provided at connection time.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:

*  Daniel Wilson

Work Items
----------

Add the "discard": True line to the Pure Storage driver.

Implement the funcionality in Nova as well.
https://review.openstack.org/#/c/205726/1


Dependencies
============

None


Testing
=======

Tempest tests will not be needed for this spec. Pure Storage CI will test
this code path with existing tempest tests.


Documentation Impact
====================

Documentation on the response to 'initialize_connection' will need to be
updated with notes explaining that this new info is provided.


References
==========

https://en.wikipedia.org/wiki/Trim_(computing)

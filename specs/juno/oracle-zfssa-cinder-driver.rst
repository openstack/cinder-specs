..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Oracle ZFS Storage Appliance iSCSI Driver
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/oracle-zfssa-cinder-driver

ZFSSA ISCSI Driver is designed for ZFS Storage Appliance product
line (ZS3-2, ZS3-4, ZS3-ES, 7420 and 7320). The driver provides the ability
to create iSCSI volumes which are exposed from the ZFS Storage Appliance
for use by VM instantiated by Openstack's Nova module.


Problem description
===================

Currently there is no support for ZFS Storage Appliance product line from
Openstack Cinder.

Proposed change
===============
iSCSI driver uses REST API to communicate out of band with the storage
controller.
The new driver would be located under cinder/volume/drivers/zfssa, and
it would be able to perform the following:

* Create/Delete Volume
* Extend Volume
* Create/Delete Snaphost
* Create Volume from Snapshot
* Delete Volume Snapshot
* Attach/Detach Volume
* Get Volume Stats

Additionaly a ZFS Storage Appliance workflow (cinder.akwf) is provided
to help the admin to setup a user and role in the appliance with enought
priviledges to do cinder operations.
Also, cinder.conf has to be configured properly with zfssa specific
properties for the driver to work.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

User will be able to use ZFS Storage Appliance product line with
Openstack Cinder.

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
  juan-c-zuluaga <juan.c.zuluaga@oracle.com>

Other contributors:
  brian-ruff <brian.ruff@oracle.com>

Work Items
----------

All the features that ZFS Storage Appliance iSCSI does.
Add CI unit test for ZFS Storage Appliance iSCSI Cinder Driver

Dependencies
============

Minimum ZFS Storage Appliance with OS8.2

Testing
=======

CI will be performed for ZFS Storage Appliance iSCSI Driver.

Documentation Impact
====================

Cinder Support Matrix should be updated.
https://wiki.openstack.org/wiki/CinderSupportMatrix


References
==========

http://www.oracle.com/us/products/servers-storage/storage/nas/overview/index.html

**ZFS Storage Appliance Workflow.**

http://docs.oracle.com/cd/E26765_01/html/E26397/maintenance__workflows.html#maintenance__workflows__bui

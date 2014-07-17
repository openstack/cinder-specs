..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cinder volume driver for X-IO ISE storage
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/xio-iscsi-fc-volume-driver

Add Cinder volume driver for X-IO ISE storage products. The driver will
implement all APIs required to support both FC and iSCSI protocols.
It communicates with the ISE storage unit using REST.

Problem description
===================

Currently no volume driver for X-IO ISE storage available in any release
branch.

Proposed change
===============

The new driver will add support for ISE storage products as backend storage
in Cinder.
The driver supports the following APIs:
* Volume Create/Delete
* Volume Attach/Detach
* Snapshot Create/Delete
* Create Volume from Snapshot
* Get Volume Stats
* Copy Image to Volume
* Copy Volume to Image
* Clone Volume
* Extend Volume

Driver will be implemented using three classes in separate files.

* class XIOISEDriver
   Main class common to FC and ISCSI driver classes.

* class XIOISCSIDriver
   Driver specific for ISCSI protocol.

* class XIOFCDriver
   Driver specific for FC protocol.

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

Addition of the X-IO volume driver will allow the end user to use X-IO storage
as backend storage in Cinder.

Performance Impact
------------------

None

Other deployer impact
---------------------

The driver can be configured with the following parameters in cinder.conf:
* san_ip - IP to REST management interface on ISE
* san_login - user name for REST management interface
* san_password - password for user
* ise_raid_level - RAID level for volumes
* ise_default_pool - storage pool to use for volume creation
* iscsi_ip_address - IP to one ISCSI target interface on ISE

The ISE ISCSI target interface specified in iscsi_ip_address will return all
target portals available on that ISE, limited to the same subnet when receiving
an ISCSI discover sendtargets request from a host identified as an Openstack
host.  This was added to allow the host to use multipathing, if enabled on the
hypervisor.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Richard Hedlind

Primary assignee:
  rhedlind

Other contributors:
  None

Work Items
----------

Common driver:
 xio_common.py
 Driver code common to FC and ISCSI.
 Done

ISCSI driver:
 xio_iscsi.py
 Driver code specific to ISCSI.
 Done. Passed driver cert test.

FC driver:
 xio_fc.py
 In progress.  Code complete, but Driver cert in progress.

Unit test:
 test_xio_fc.py
 test_xio_iscsi.py
 In progress.

CI environment will be setup, one for each driver type.

Dependencies
============

None

Testing
=======

Test using existing test infrastructure according to driver submission steps.

Documentation Impact
====================

Support Matrix needs to be updated to include X-IO support.
https://wiki.openstack.org/wiki/CinderSupportMatrix

Block storage documentation needs to be updated to include X-IO volume driver
information in the volume drivers section.
http://docs.openstack.org/

References
==========

None

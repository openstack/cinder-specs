..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Integrate VMDK driver with oslo.vmware library
============================================================

https://blueprints.launchpad.net/cinder/+spec/vmdk-oslo.vmware

The common code between various VMware drivers was moved to the oslo.vmware
library during Icehouse release. The VMDK driver should be updated to use
this library.

Problem description
===================

The oslo.vmware library (https://github.com/openstack/oslo.vmware) contains
code for invoking VIM/SPBM APIs, session management, API retry and
upload/download of virtual disks. The VMware drivers for nova, glance and
ceilometer have already integrated with oslo.vmware. This spec proposes
the integration of VMDK driver with oslo.vmware.

Use Cases
=========

Proposed change
===============

* Changes are mostly replacing import statements for the following modules:

  * Replace api with oslo.vmware.api
  * Replace vim with oslo.vmware.vim
  * Replace pbm with oslo.vmware.pbm
  * Replace io_util with oslo.vmware.image_transfer
  * Replace vmware_images with oslo.vmware.image_transfer
  * Replace read_write_util with oslo.vmware.rw_handles

* Remove duplicate exceptions in error_util and use oslo.vmware.exceptions

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

None

Performance Impact
------------------

None

Other deployer impact
---------------------

The oslo.vmware version mentioned in the requirements file needs to be
installed.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  vbala <vbala@vmware.com>

Other contributors:
  None

Work Items
----------

* Add dependency on oslo.vmware and replace import statements
* Remove duplicate exceptions and use the ones defined in oslo.vmware
* Delete unused modules including their unit tests

Dependencies
============

None


Testing
=======

Unit tests for the duplicate modules will be removed. There won't be any new
tests as the changes are purely code reorganization.

Documentation Impact
====================

None

References
==========

None

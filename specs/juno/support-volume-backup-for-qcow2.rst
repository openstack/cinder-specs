..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Support volume backup for qcow2
================================

https://blueprints.launchpad.net/cinder/+spec/support-volume-backup-for-qcow2

Currently, cinder-backup doesn't support qcow2 format disk. Add the support
for it will make drivers which use qcow2 as volume, such as glusterfs etc,
work together with cinder-backup, and can also make nfs driver use qcow2 as
volume be possible.

Problem description
===================

Currently, cinder-backup doesn't support qcow2 format disk because the backup
code assumes the source volume is a raw volume. The destination (i.e. swift,
rbd) should absolutely remain universal across all volume back-ends.

Proposed change
===============

* Add qemu-nbd support to cinder-backup. Qemu-nbd can mount qcow2 volume as
  a raw device to the host
* The backup_volume method in base class of remotefs driver (cinder.volume.
  drivers.nfs.RemoteFsDriver:backup_volume) will mount qcow2 volume as nbd
  device before call backup_service's backup method

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

Storage node which running cinder-volume will contains nbd kernel module.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Trump.Zhang <zhangleiqiang@huawei.com>

Work Items
----------

* Add qemu-nbd support to cinder-backup. Qemu-nbd can mount qcow2 volume as
  a raw device to the host
* The backup_volume method in base class of remotefs driver (cinder.volume.
  drivers.nfs.RemoteFsDriver:backup_volume) will mount qcow2 volume as nbd
  device before call backup_service's backup method


Dependencies
============

None


Testing
=======

None


Documentation Impact
====================

None


References
==========

None

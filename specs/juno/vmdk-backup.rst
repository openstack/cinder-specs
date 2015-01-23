..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Support backup and restore of volumes created by VMDK driver
============================================================

https://blueprints.launchpad.net/cinder/+spec/vmdk-backup

The volumes created by the VMDK driver are virtual disks stored in datastores
managed by ESX or vCenter server. Currently, the ``backup-create`` and
``backup-restore`` operations are not supported for these volumes. This
blueprint proposes adding support for these operations in VMDK driver.

Problem description
===================

The default implementation of ``backup-create``\\ ``backup-restore`` does the
following steps:

* Attach the volume as a block device or file.

* Backup\\restore the file by calling backup service.

* Detach the volume.

It uses  an instance of ``InitiatorConnector`` (determined by the back-end
driver protocol) to do the actual\\detach. There is no  ``InitiatorConnector``
for the ``vmdk`` protocol and hence the attach\\detach fails for volumes
created by the VMDK driver. This blueprint proposes adding support for
``backup-create``\\ ``backup-restore`` for these volumes.

Use Cases
=========

Proposed change
===============

The change involves overriding the default implementations of ``backup_volume``
and ``restore_backup`` methods in ``VMwareEsxVmdkDriver``. The steps in
``backup_volume`` are listed below:

* Create the backing VM if it not found.

* Download the stream-optimized version of the virtual disk corresponding to
  the volume to a temporary directory.

* Call ``backup_service.backup()`` method to backup the stream-optimized
  virtual disk file.

* Delete the temporary file.

Following are the steps in ``restore_backup``:

* Call ``backup_service.restore()`` to download the stream-optimized virtual
  disk file to a temporary directory.

* If the backing VM doesn't exist (in the case of restoring the backup to
  create a new volume), import the stream-optimized virtual disk file to create
  a new backing VM.

* If the backing VM exists, import the stream-optimized virtual disk file to
  create a temporary VM and reconfigure the backing VM to replace its virtual
  disk with that of the temporary VM.

* Delete the temporary file and temporary VM.

Alternatives
------------

**HTTP read/write**: It is possible to create an HTTP connection to read/write
from/to a virtual disk file in vCenter/ESX and an adapter can be written for
this connection to support some of the file operations required by the backup
drivers. This implementation works for both Swift and Ceph backup drivers. But
the TSM backup driver raises ``InvalidBackup`` exception if the volume to be
backed up is not a block device or regular file.

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

None

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

* ``backup_volume`` method
* ``restore_backup`` method

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

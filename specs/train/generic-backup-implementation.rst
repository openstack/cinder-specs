..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Generic backup implementation
=============================

https://blueprints.launchpad.net/cinder/+spec/generic-backup-implementation

Generic backup implementation will allow us to use any supported volumes
backend as backup storage. So we won't need to implement specific backup driver
for supported backends.


Problem description
===================

We've got different backup and volume drivers now. That's why we have to
re-implement the part of volume driver as backup driver if we want to use
the same storage for backups.

Use Cases
=========

1. Be able to use any cinder-supported storage backend for backups.

Proposed change
===============

Implmement generic backup implementation in a base volume driver like we did
for volume migrations. After this vendors could easily implement
storage-assistant backups for their drivers.

We will have base backup drivers class for storages like Swift, Google Cloud
Storage, etc which will implement only backup-related features.

It will allow a volume backed up to storage A to be restored to a volume
on any storage B.

Cinder should not allow to use the same storage both for volume and backups.
In such case, backup driver initialization should fail. If storage supports
different pools Cinder will allow to create backup on the same storage but in
the different pool.

We don't need to have 'backup and volumes in the same storage' feature in
Cinder even configurable because we can use snapshots or clone volume for it.
Backups should be in different storage or at least in another pool.

As a generic implementation, cinder will use the same mechanism as generic
volume migraion:

* create volume on a destination storage
* attach both source and destination volumes to a cinder node
* use 'dd' tool to copy volume data
* detach both volumes from a cinder node

In case if volume is 'in-use' we'll create temporary snapshot and do backup
from it.

We can't use 'clone volume' feature between different storages.

Vendor-specific changes
-----------------------
Vendors or drivers maintainers could implement vendor-specific backup
implementation to use storage API for faster backup process.

Alternatives
------------

Follow the current approach with separate volumes and backups drivers.

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

Backups are likely to be faster to block backends than to swift. It means
backup to block storage could be faster than a backup to object storage.


Other deployer impact
---------------------

Operator should be able to configure backups storage and cinder backup driver.

Developer impact
----------------

* Volume drivers could implement vendor-specific backup implementation


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  Ivan Kolodyazhny <e0ne@e0ne.info>

Other contributors:
  Volume Driver maintainers

Work Items
----------

TDB


Dependencies
============

None


Testing
=======

* Unit tests
* Tempest tests should be implemented in a new feature group


Documentation Impact
====================

Operators documentation should be updated according to spec implementation.


References
==========

* http://eavesdrop.openstack.org/meetings/cinder/2016/cinder.2016-08-10-15.59.html

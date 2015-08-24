..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Efficient volume copy for volume migration
==========================================

https://blueprints.launchpad.net/cinder/+spec/efficient-volume-copy-for-cinder-assisted-migration

Currently, cinder supports these 3 ways for volume migration.

 +-+--------------------+---------------------------+----------------------+
 |#| Migration method   | Data copy method          | Handler of migration |
 +=+====================+===========================+======================+
 |1| storage driver     | depend on vendor storage  | inside storage array |
 +-+--------------------+---------------------------+----------------------+
 |2| copy_volume_data() | dd command                | cinder volume node   |
 +-+--------------------+---------------------------+----------------------+
 |3| swap_volume()      | libvirt block rebase copy | nova compute node    |
 +-+--------------------+---------------------------+----------------------+

During volume migration case #2, Cinder uses dd command for data copy
of volume migration, but the copy always copies full blocks even if
the source data contains many null and zero blocks.
The dd command has an option conv=sparse to skip null or zero blocks for
more efficient data copy.

The purpose of this request is to introduce a mechanism to handle sparse
copy option properly for volume migration using dd command.

Problem description
===================

If we create a volume from thin-provisioning pool, volume blocks are
not pre-allocated, and the volume blocks will be allocated on demand.
In this situation, if we migrate detached volume using dd command,
dd will copy full block from source to destination volume even if
the source volume contains many null or zero blocks. As a result,
usage of the destination volume will be always 100%. Here is an example
volume migration using thin LVM driver.

* Before migration

::

  LV            VG    Attr       LSize   Pool     Origin Data%
  vg1-pool      vg1   twi-a-tz--   3.80g                10.28
  volume-1234   vg1   Vwi-a-tz--   1.00g vg1-pool       19.53

* After migration without conv=sparse option

::

  LV            VG    Attr       LSize   Pool     Origin Data%
  vg2-pool      vg2   twi-a-tz--   3.80g                31.45
  volume-1234   vg2   Vwi-a-tz--   1.00g vg2-pool      100.00

Use Cases
=========

Using sparse copy is able to reduce volume usage of destination storage
array compared to using full block copy.

Proposed change
===============

If the volume pre-initilization(zero cleared) is ensured beforehand
at the destination c-vol, we can skip copy of null and zero blocks
to destination volume by using conv=sparse option for dd.

However, if the destination volume is not zero cleared beforehand,
we should disable sparse copy and copy full block from source to
destination volume to avoid data corruption problem.

For example, following case, we should disable sparse copy and copy
full block because new volume of thick LVM might not be initialized,
we must not use conv=sparse option for similar cases.

* Source c-vol driver: Thin LVM
* Dest c-vol driver: Drivers who provides an uninitialized new volume.
  (ex. Thick LVM)

In order to handle sparse option properly, following changes
are required.

1. Add cinder_sparse_copy_volume capability into capabilities list
   as a well-defined option.

::

  'cinder_sparse_copy_volume': {
     'default': 'False',
     options: {}
  },

2. Get capability list from the "destination cinder volume driver"
   using ``get_capabilities`` method before starting volume copy.

``get_capabilities`` method is proposed following Spec.

* Updating Get Volume Driver Capabilities Spec:
  https://review.openstack.org/#/c/183947/1

These are steps of migrate_volume related to sparse copy.

::

  source c-vol driver                   dest c-vol driver
   |                                           |
   | ========================================> |
   |   Ask capability of sparse copy via       |
   |   get_capabilities                        |
   | <======================================== |
   |   Dest c-vol returns capabilities to      |
   |   source c-vol                            |
   | ========================================> |
   | Start volume copy with or without sparse  |
   | based on the retun from dest c-vol        |
   |                                           |


Alternatives
------------

Copy full blocks from source volume to destination volume. And then,
try fstrim or blkdiscard to release unused blocks to backend storage.
However, if the backend storage doesn't support UNMAP SCSI command,
we can't release unused blocks after migration.

Data model impact
-----------------

None

REST API impact
---------------

Add cinder_sparse_copy_volume parameter as a well_defined capability.

::

  'cinder_sparse_copy_volume': {
     'default': 'False',
     options: {}
  },

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

* This feature improves required time to copy data between source and
  destination volume.

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
  mitsuhiro-tanino

Work Items
----------

* Add cinder_sparse_copy_volume capability into capabilities list.
* Add handler of get_capabilities in migrate_volume method.
* Update LVM reference implementation with cinder_sparse_copy_volume
  capability.
* Update NFS driver with cinder_sparse_copy_volume capability:

  NFS driver already has _sparse_copy_volume_data option but this way
  doesn't ask capability to destination cinder volume before volume copy.
  We should use same way to handle sparse copy.


Dependencies
============

None

Testing
=======

* Unit tests

Documentation Impact
====================

* Need to update openstack cloud administrator guide.
  http://docs.openstack.org/admin-guide/blockstorage_volume_migration.html

References
==========

* Related Cinder Spec

  Updating Get Volume Driver Capabilities Spec:
  https://review.openstack.org/#/c/183947/1

* Related fix

  https://review.openstack.org/#/c/183633/

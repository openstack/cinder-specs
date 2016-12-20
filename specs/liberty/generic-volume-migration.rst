..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Generic Volume Migration
========================

https://blueprints.launchpad.net/cinder/+spec/generic-volume-migration

The intent is to allow volume drivers that do not support iSCSI and use other
means for data transport to also participate in volume migration operations.
At the moment, the existing code uses create_export to create and attach
a volume via iSCSI to perform I/O operations.  By making this more generic, we
can allow other drivers to take part in volume migration as well.

This change is necessary to support volume migration for the Ceph driver.

Problem description
===================

When migrating a volume between two backends, the copy_volume_data routine in
the source volume's driver is executed to move the blocks from one volume to
another. This routine assumes that both source and destination volumes can be
attached locally via iSCSI and calls create_export to attach the volume to the
local cinder-volume instance.  This is technically not necessary for local
volumes and also prevents drivers such as Ceph from participating in volume
migration operations.

Use Cases
=========

Proposed change
===============

By using a file-like object abstraction similar to the backup volume driver
approach, we can allow the volume driver to determine how best to attach its
volume locally.  For most drivers, this will continue to be iSCSI.  And for
Ceph, a RBD object that support file semantics is returned.  In both cases, the
copy_volume_data routine operates on these file objects as it does now, but
without explicit knowledge of the underlying transport mechanism.

Specifically, I would like to add a routine to the volume driver
(cinder/volume/driver.py):

::
    open_volume()
    close_volume()

These routines are called when doing local volume attachment as in
copy_volume_data().  I think these routines are necessary because local file
I/O operations may require different transport than the standard
create_export/initialize_connection approach.  In the case of Ceph,
create_export does nothing and initialize_connection returns the necessary RBD
connection info for Nova to communicate with the Ceph volume directly.  We want
driver authors to retain this flexibility.

The open_volume routine will return a volume file object that supports file
operations such as read, write, seek, etc.  The close_volume routine will
handle teardown and cleanup of the volume file object.

This will allow Ceph to return a file-like object that communicates directly
with the volume.  Other backends, such as LVM, will continue to create an iSCSI
export, attach it, open the resulting block device, and return a handle to this
file.  Additionally, local LVM volumes could skip the iSCSI step and open the
local block device directly.  This change could be made later, this effort is
to make the minimum amount of changes necessary to support the feature.

With these driver routines in place, copy_volume_data is modified to make use
of these calls instead of the using _attach_volume directly.  For non-Ceph
drivers the result is exactly the same as it was.  And the RBD driver can now
implement these routines to add support for volume migration.

Alternatives
------------

The most obvious alternative is to change the Ceph driver to implement the
export routines and allow volume attachment via iSCSI.  This would mean using
the rbd kernel driver which is not ideal, as it introduces code into the
running kernel on the cinder-volume node and lags a bit in features as compared
to librbd.  We would like to avoid this if at all possible.

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

If everything works correctly, volume migration between Ceph and some other
backend should function as expected.  In both directions.

Performance Impact
------------------

The additional abstraction only enables alternatives to iSCSI attachment, no
impact on existing functionality is introduced.

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
  jbernard

Work Items
----------

1. Introduce the file-object abstraction and change existing code to use iSCSI
   for volume attachment - just as it does now.  Ceph migration will continue
   to fail.

2. Implement the Ceph-specific attach logic, that should allow volume migration
   to succeed between other backends.

3. Add test(s) to tempest to verify code paths are executed correctly and yield
   a block-for-block identicaly migration result.

Dependencies
============

None

Testing
=======

I don't think volume migration is covered in the gate, but this could be tested
in tempest.  Ceph volume migration to a non-Ceph backend should be successful
in both directions.  If that test passes, this effort was a success.

Documentation Impact
====================

I'm don't think anything is necessary here.

References
==========

None

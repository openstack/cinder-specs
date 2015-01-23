..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
NFS Snapshots
==========================================

https://blueprints.launchpad.net/cinder/+spec/nfs-snapshots

Add support for snapshots (create, delete, clone from) to the
NFS volume driver, so that it is as functional as other
Cinder drivers.

Problem description
===================

The NFS driver does not support many features expected in a modern Cinder
driver.

* create snapshot
* delete snapshot
* clone from snapshot

Use Cases
=========

Proposed change
===============

The basic model for how snapshots can be implemented here has been proven
in the GlusterFS driver.  The general idea here is to re-use that same
model and much of the code for the NFS driver, which works the same as
the GlusterFS driver in general.

In Juno, most of the snapshot-related code was refactored into a
RemoteFSSnapshot class.

The NFS driver should inherit code from this base class (and probably
make minor changes) to gain snapshot support.

Alternatives
------------

No real alternative, baseline NFS servers do not have the ability to snapshot
files.  This method is already used in other Cinder drivers and works, so the
only alternative is to decide we don't want snapshots for this driver.

Data model impact
-----------------

None

REST API impact
---------------

Nothing significant.

initialize_connection will return additional fields specifying the format
of the volume file so that Nova knows how to attach it properly.

Security impact
---------------

None, see notes here.

This change needs to be tested with the NFS security infrastructure added
in Kilo to ensure nothing breaks and that all files before, after, and during
snapshot operations have the expected permissions.  There is no reason to
expect problems here.

In Juno a security concern related to qcow2 file handling was addressed.
( https://bugs.launchpad.net/cinder/+bug/1350504 )

This is now considered fixed from a security perspective as there are no
known ways to exploit it, but is still being fixed in a more robust manner
in the "Support storing volume format info" spec.  This work is possible in
parallel as there is no strict interdependency here.
( https://review.openstack.org/#/c/103750/ )

Notifications impact
--------------------

None

Other end user impact
---------------------

We do not yet support backups of qcow2-based volumes.  (Unsure if targeted
in Kilo elsewhere.)

Performance Impact
------------------

The NFS driver needs to have a per-volume-id lock for operations that
manipulate volume/snapshot data.  This does not cause any performance
regression in existing functionality and should not be a large concern
in most scenarios when using snapshots.

See https://review.openstack.org/#/c/106095/ (GlusterFS example)

Other deployer impact
---------------------

New config option, nfs_qcow2_volumes, to specify if you would like to
always create volume files as qcow2 images rather than raw.

Developer impact
----------------


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eharney and akerr (?)

Other contributors:
  NetApp has indicated they will CI this driver.  (bswartz)

Work Items
----------

- Try to inherit RemoteFSSnapshot code into NFS driver.
- Iterate on issues there.
- Test security concerns.


Dependencies
============

Storing volume format info: https://review.openstack.org/#/c/103750/
 - Not a strict dependency, and which lands first is not vital, but this
   effort also needs to land in Kilo along with this work and is strongly
   related.

I will submit another spec addressing the snapshot API between Cinder and
Nova which will be used here, but nothing there blocks this work.


Testing
=======

Standard third-party CI of the NFS driver

Consider testing this in-gate as a later effort


Documentation Impact
====================

New config option, nfs_qcow2_volumes.


References
==========

Original snapshot effort which this is based on:
 https://blueprints.launchpad.net/cinder/+spec/qemu-assisted-snapshots

RemoteFS snapshot code refactoring, done to support this effort:
 https://review.openstack.org/#/c/106066/

SMBFS volume driver, an example of another driver re-using this code:
 https://review.openstack.org/#/c/106046/


..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Windows SMB Volume Driver
=========================

https://blueprints.launchpad.net/cinder/+spec/hyper-v-smbfs-volume-driver

Currently, there are Cinder volume drivers which make use of network-attached
storage file systems such as GlusterFS of NFS. The purpose of this blueprint
is adding a volume driver supporting SMB based volume backends.

This driver will support Windows Server 2012 (R2 included).

Problem description
===================

SMB is a widely used protocol, especially in the Microsoft world. Its
simplicity along with the big improvements that were introduced in SMB 3
make this type of volume backend a very good alternative.

SMB 3 brings features such as transparent failover, multichanneling using
multiple NICs, encrypted communication, and RDMA.

This driver will be backwards compatible, supporting older versions of SMB.
It will support using any type of SMB share, including:

    - from Scale-Out file servers to basic Windows shares;

    - Linux Samba shares;

    - vendor specific hardware exporting SMB shares.

Use Cases
=========

Proposed change
===============

This driver will include all the features required by the Juno release. It
will have a similar flow with other netowrk attached file system drivers,
managing images regarded as volumes while hosted on SMB shares.

It will include support for vhd and vhdx images, which will be hosted on
SMB shares. The driver will support password based authentication for
mounting the SMB shares.

The snapshot management will be done in a similar way as the Gluster driver,
using an additional ".info" file containing mappings between snapshot ids and
their actual paths. This is required as the path can change in time (for
example when deleting a snapshot from the chain, the backing file will be
changed). Differencing vhd/vhdx images will be used as backing files.

This driver will be based on the SMB volume driver which will be available
for Linux distributions.

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

The driver will periodically ensure that the chosen shares are mounted and also
retrieve informations such as free space or total allocated space.

Certain volume related operations will require to be synchronized.

Other deployer impact
---------------------

The user will provide a list of SMB shares on which volumes may reside. This
list will be placed in a file located at a path configured in the cinder config
file.  This share list may contain SMB mount options such as flags or
credentials.

The config file will also contain the path to the Samba config file. Oversubmit
and used space ratios may also be configured.

The user will be able to choose whether to use vhd or vhdx images. The volume
type may be parsed as extra spec when creating a volume.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <lpetrut@cloudbasesolutions.com>

Work Items
----------

Provide volume related operations support using images hosted on SMB shares.

Override the Linux SMB volume driver methods that are platform specific.

Dependencies
============

Libvirt smbfs volume driver blueprint:
 https://blueprints.launchpad.net/nova/+spec/libvirt-smbfs-volume-support

Hyper-V smbfs volume driver blueprint:
 https://blueprints.launchpad.net/nova/+spec/hyper-v-smbfs-volume-support

Linux smbfs volume driver blueprint:
 https://blueprints.launchpad.net/cinder/+spec/smbfs-volume-driver

Testing
=======

A Cinder CI will be testing the SMB related features.

Documentation Impact
====================

Using the SMB backend will be documented.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
NFS backup driver for Cinder
============================

https://blueprints.launchpad.net/cinder/+spec/nfs-backup [1]

Currently, cinder-backup doesn't support backup to an NFS-supplied
data repository.  This blueprint is for work to make this possible.

Problem description
===================

* Currently backup of cinder volumes is available to  or Ceph
  object stores, or via Tivoli Storage Manager.  These choices, while
  appropriate for some, don't address all OpenStack operators or
  deployment scenarios.

* Some OpenStack administrators who own NFS storage devices or scalable
  NFS storage clusters want to leverage these to provide backup
  for their cinder volumes, irrespective of whether NFS is used as
  a cinder volume backend.

* Some backend providers need dynamic mount management from OpenStack
  as in the NFS volume drivers in order to better manage backup data
  security, to provide a more deterministic environment for support
  and service contracts, and to minimize dependencies external to
  OpenStack distributions and automation packages.

* The NFS backup driver should not replicate functionality provided
  by the POSIX filesystem driver (currently in review, see [3]
  and [4]).

Use Cases
=========

Proposed change
===============

The following proposal presupposes a context in which the POSIX
filesystem driver ([3] and [4]) has been merged upstream and that
the POSIX filesystem driver, in turn, extends a "chunking" base
class that abstracts common functionality previously only in the
Swift backup driver (see [5] and [6]).

We propose to implement a NFSBackupDriver class under the
POSIX filesystem BackupDriver class::

                  +----------------------+
                  |                      |
        +-------->|     BackupDriver     |<-----+
        |         |                      |      |
        |         +----------^-----------+      |
        |                    |                  |
        |                    |                  |
  +-----+------+ +-----------+---------+ +------+-----+
  |    TSM     | |        Chunking     | |    Ceph    |
  | BkupDriver | |       BkupDriver    | | BkupDriver |
  |            | |          Base       | |            |
  +------------+ +---^--------------+--+ +------------+
                     |              |
                     |              |
               +-----+-----+  +-----+------+
               |   Swift   |  |   POSIX    |
               | BkupDriver|  | BkupDriver |
               | (default) |  |            |
               +-----------+  +-----^------+
                                    |
                                    |
                              +-----+------+
                              |    NFS     |
                              | BkupDriver |
                              |            |
                              +------------+


The NFS Backup Driver will override the parent POSIX file system
driver's __init__() method in order to do dynamic mount management
using "brick" code shared with the volume driver for this purpose, as
in [2].  With this extension, it will connect to an externally
provisioned NFS share at initialization.

This generic NFS Backup Driver will otherwise just inherit methods
to create, delete, and restore from backups from its parent.

Backend vendors who can add value by, e.g. data transfer techniques
that avoid or reduce transfer operations between the cinder node and
the backend storage array can themselves extend the generic NFS Backup
Driver. To this end, we have agreed that the POSIX backup driver will
explicitly implement a generic data transfer method which subclasses
can override, or will inherit this generic method from the Chunking
Base Class. Moreover, while this generic data transfer method can
implement compression as an option, it is critical that it be only an
option since backends may choose to implement compression or
deduplication themselves.

A benefit of the approach presented here is that a user should be able
to switch between posix, NFS, and vendor-specific extensions of the
NFS backup driver, and backup operations will continue to work even
though they use different data transfer mechanisms.

Alternatives
------------

In the diagram, the common code shared by Swift and POSIX is
implemented via inheritance and an "is-a" relationship.  A "has-a"
relationship to a library would also work for this proposal.

If other NAS drivers (GlusterFS, SMB, etc.) elect to also implement
dynamic mount management, we can abstract out a RemoteFS Backup Driver
as their common parent and interpolate it between the NFS backup
driver (and other NAS drivers) and the POSIX filesystem driver.  Or we
may be able to just share most configuration options and promote the
original NFS Backup Driver implementation to the RemoteFS driver role.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

NFS backups will run as a non-root user and the proposed driver will
not elevate user privileges except when it mounts the backup
repository.  When it does the mount, it will invoke brick code to do
so.  Thus the only privilege elevation from this driver will be in a
common library.  That library has been audited and tweaked as
part of recent NFS security enhancements [7], addressing the NFS side
of launchpad bug 1260679 [8].

Because the backup driver runs as a regular user it works with secure
NAS environments in which root squash is enabled.

The backup driver will chmod '660' actual backup data and metadata
and chmod '770' directories in the path to same.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Backup service can in general have performance impact.  Future
enhancements to or extensions of this driver class could seek to
reduce data transfer bandwidth during backup and restore or pursue
differential backup strategies to reduce the amount of work involved
in typical backups.

POSIX path backups can potentially produce directories containing
a large number of backup files, such that directory operations could
be very costly.  We should implement some directory hierarchy to keep
directories sized well.  See [2] for one way to do this.


Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Implementation
==============

Backup data and metadata will be written to the repository at unique
paths that are a function of the backup ID.  That path will be stored
in the backup record service_metatdata so that it can be used to
navigate to the backup data and metadata for restore and delete
operations.

Backup and restore operations will share a common method for transfer
of data from volume to backup and vice versa.  This method will be
initially implemented as a naive block copy but will allow for
enhancement or extension to e.g. reduce or eliminate data movement
between the cinder node and the NFS server for the backup repository.


Assignee(s)
-----------

Primary assignee:
  Tom Barron (tbarron)

Other contributors:
  Kevin Fox (kfox1111) - POSIX filesystem driver and Chunking Base class

Work Items
----------


Dependencies
============

None

Testing
=======

* Appropriate unit tests will be added.
* Existing tests with backup_driver option in cinder.conf set
  for the NFS driver rather than for Swift will provide tempest coverage.

Documentation Impact
====================

Update the backup section of the OpenStack Configuration Reference to indicate
how to perform volume backups using an NFS server.  Specifically, document:

* New value for cinder.conf backup_driver option: cinder.backup.drivers.nfs

* New cinder.conf option 'backup_nfs_share' with default value None and values
  in one of the following formats::

  - <fqdn>:<posix-path>
  - <ipv4addr>:<posix-path>
  - [<ipv6addr>]:<posix-path>

* New cinder.conf option 'backup_nfs_mount_options' with default value None
  and values as specified in NFS man pages and as used in the NFS volume
  driver.  Note: it may make sense to set the default to values tuned for
  backup performance rather than leaving the default None if we can agree
  on such values for NFS common.  Otherwise, different NFS backends will
  likely want to extend this class and set optimal backend-specific default
  options.


References
==========
[1]: https://blueprints.launchpad.net/cinder/+spec/nfs-backup
[2]: https://review.openstack.org/#/c/138234
[3]: https://blueprints.launchpad.net/cinder/+spec/add-backup-driver-nas-storage
[4]: https://review.openstack.org/#/c/82996
[5]: https://blueprints.launchpad.net/cinder/+spec/chunked-backup-base-class
[6]: https://review.openstack.org/#/c/139737/
[7]: https://review.openstack.org/#/c/107693
[8]: https://bugs.launchpad.net/cinder/+bug/1260679


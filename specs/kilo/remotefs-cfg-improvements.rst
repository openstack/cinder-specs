..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
RemoteFS Config Improvements
==========================================

https://blueprints.launchpad.net/cinder/+spec/remotefs-share-cfg-improvements

RemoteFS drivers (NFS, GlusterFS, etc.) are currently configured by adding
a list of shares to a text config file which is referenced by cinder.conf.
This means that one driver instance manages a handful of storage locations
for the driver.  This work will a) have these drivers configured like most
other Cinder drivers are, and b) leverage the Cinder scheduler for selection
between different storage backends rather than having the driver act as a
pseudo-scheduler.


Problem description
===================

The configuration system for NFS/GlusterFS/etc drivers:
 * is different from other drivers
 * is more complex than necessary
 * limits functionality such as migration

Use Cases
=========

Proposed change
===============

Replace the <x>_shares_config setting with settings that can be used to login
to the storage platforms.  This means that an nfs_shares_config file such as::

    192.168.1.10:/export1 -o sync
    192.168.1.11:/export2 -o vers=nfs4

would become, in cinder.conf::

    [nfs1]
    address = 192.168.1.10
    export_path = /export1
    options = -o sync

    [nfs2]
    address = 192.168.1.11
    export_path = /export2
    options = -o vers=nfs4

Each Cinder backend will then only manage one export rather than a handful of
exports.  This brings the RemoteFS drivers closer to how other Cinder
drivers operate.

Alternatives
------------

Leave things as they are today.  (Not desirable.)

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

It will be possible to use Cinder volume migration to move volumes between
all NFS exports when previously this was not always possible (since the
different exports were managed by the same driver instance).

Performance Impact
------------------

None

Other deployer impact
---------------------

nfs_shares_config, glusterfs_shares_config, etc., will be deprecated
(but still functional for Kilo).   Setting the new options will cause
these settings to be ignored.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eharney

Other contributors:
  Other interested parties?

Work Items
----------

* Create new options for address, export, mount options
* Mark options and code to be removed in L as deprecated in Kilo

Dependencies
============

None


Testing
=======

The NFS driver and GlusterFS drivers will be gaining CI during the Kilo
cycle which will cover this.

Manual testing should cover both the current and new configuration paths.

Documentation Impact
====================

New configuration options and possibly guide changes for configuring the NFS
driver.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Volume Rekey
==========================================

https://blueprints.launchpad.net/cinder/+spec/volume-rekey

Cinder supports volume encryption with keys stored in Barbican.

This spec tracks some improvements we can make in how we handle
encryption keys.


Problem description
===================

When cloning volumes, cinder clones the encryption key as well,
so multiple volumes are unlockable with the same encryption key.

We can make this encryption scheme more robust by changing the
encryption key upon volume clone, so that cloned volumes are not
unlockable with the key of the source volume.

This is possible since we encrypt volumes using LUKS. Changing the
key does not require re-encrypting the volume.

Use Cases
=========

Security hardening for volume encryption.

Proposed change
===============

When a volume is cloned, attach it as part of the clone process,
and change the encryption key using LUKS tools.

The rest of the clone process continues as normal afterward.

My current implementation does this by calling a new
rekey_volume() method in the create_volume flow, which uses
"cryptsetup luksChangeKey".  This should work for any iSCSI/FC
drivers, which already must perform a similar attachment when
creating a volume from an image.

Some work (planned for after Train) is still needed to make this
work for RBD, because there does not seem to be a qemu-img tool
that can change encryption keys, and cryptsetup requires a local
block device.  This leaves us with two options for RBD:
a) use krbd mapping to get a block device
b) use rbd-nbd to get a block device

NBD is not widely supported in relevant OSes, so krbd looks like
the choice there.

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

* Volume encryption is better hardened against threats due to
  compromise of a single encryption key.


Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Slightly more time to clone encrypted volumes.

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
  eharney

Work Items
----------

* Implement this for iSCSI/FC drivers
* Test with the LVM driver

In a later release...
* Implement this for RBD
- Requires some additional effort
* Consider additional cases where this concept would be useful
- Volume transfer
- Backup restoration (?)


Dependencies
============

None


Testing
=======

Will be on by default and therefore tested by tempest tests
that clone encrypted volumes.

Documentation Impact
====================

None

References
==========

None

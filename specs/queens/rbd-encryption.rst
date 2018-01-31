..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
RBD Volume Encryption
==========================================

https://blueprints.launchpad.net/nova/+spec/libvirt-qemu-native-luks

This feature adds support to the Cinder RBD volume driver
to support Cinder's volume encryption.

This requires a few changes in Cinder and Nova due to the fact that
RBD volumes are attached by qemu directly and not as block devices
on the host.

This fills a feature gap for the RBD driver in Cinder.


Problem description
===================

The RBD driver does not support volume encryption.

Use Cases
=========

Volume encryption is a common requirement for deployments,
particularly where a deployer needs to meet particular security
standards.

Proposed change
===============

Enable volume encryption for RBD via qemu's LUKS block layer.

This means that Nova has to support libvirt operations to manage
this qemu feature.  This is done here:

* https://review.openstack.org/#/c/523958/

We also need Cinder to format volumes upon creation with a LUKS
header.  This is currently done by os-brick for iSCSI drivers,
but can't be done in the same way for RBD since there is no
block device on the compute host, and dm-crypt is not used.

(Note: this will also be true when using qemu's iSCSI initiator
with Nova)

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

This is a security-focused feature, but it uses the already existing
infrastructure of Cinder volume encryption.

The way encryption works when using RBD is slightly different from
other Cinder drivers.  Decryption/encryption is handled inside of
qemu rather than at the device-mapper layer on the host via dm-crypt.

This means fewer operations having to be run as root, and less exposure
of decrypted data to the rest of the system via block devices.

But, the feature in general has the same security implications as
cinder volume encryption does for other drivers.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Using encryption could result in slightly higher CPU usage on compute
nodes.  Should be comparable to using encryption with any other Cinder
driver.

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

Other contributors:
  lyarwood

Work Items
----------

* https://review.openstack.org/534811/
* https://review.openstack.org/523958/

Dependencies
============

* Nova changes here:
  - https://blueprints.launchpad.net/nova/+spec/libvirt-qemu-native-luks

* QEMU 2.6
* libvirt 2.2.0

Testing
=======

This feature will be covered by the standard tempest tests used for all
volume drivers.

Gate configuration issues are being sorted out here:
  https://review.openstack.org/#/c/536350/


Documentation Impact
====================

* Document that volume encryption now works for the RBD volume driver
* Current limitation: attached volume migration is not supported

References
==========

* https://review.openstack.org/#/q/topic:bp/libvirt-qemu-native-luks

* https://blueprints.launchpad.net/nova/+spec/libvirt-qemu-native-luks

* http://lists.openstack.org/pipermail/openstack-dev/2018-January/126440.html

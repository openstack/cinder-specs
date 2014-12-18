..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Native Rest Block Driver
==========================================

https://blueprints.launchpad.net/cinder/+spec/restblock-driver

We, at scality, are working on a Linux native kernel driver: Rest Block Driver.
(it is on github, but still private as we want to reach a certain point before
releasing it). This is a kernel driver that is able to talk to a REST-based
storage service (we plan to eventually support multiple rest backends), in
order to attach the files from that storage as native block devices on the host
linux. One of the key points is that it allows a REST-based volume storage to
take advantage of linux's block driver cache.

Problem description
===================

As a company, we are interested in providing an Openstack integration to our
clients, and as such, we want to provide a new driver for cinder to drive and
make use of the aforementionned Rest Block Driver.

This block driver currently offers the following simple features:

 * Mirror-server list management (failover and load balancing)

 * Provisionning (create/extend/delete)

 * Automatic attach/detach of volumes (practical for use within cinder)

Also, we have plans for a few features that should come shortly:

 * Storage Pool management to provide volumes from different storage pools.
   Each pool will have their own list of mirrors to choose from.

 * Real failover implementation

 * Multiple back-ends support (for multiple REST-like protocols
   implementations, ours being a CDMI implementation)

 * Native snapshot management

Proposed change
===============

The Rest Block Driver is mostly controlled through the /sys file system. Thus,
most of the control operations will be implemented using the process utils
module. For the general feature implementation, here is how we plan to support
the multiple required features of a Cinder driver:

 * Provisionning (create/extend/delete) : natively supported by the kernel
   driver

 * Automatic attach of volumes at setup-time: Natively supported by the kernel
   driver

 * Snapshots (create/delete) : Supported through LVM tool classes/functions

 * Copy To/From Image: Supported thanks to image_utils

 * Volume Cloning (from volume or from snapshot): Supported through copy utils

Please not that this reflects the current state of the kernel driver, and might
evolve.

In the future it will provide a way to manage the snapshots from the driver
itself, but it is not yet supported. Thus, the driver will use the lvm code
tools to manage thin snapshot provisionning and support snapshots for the
volumes exposed by our Rest Block Driver.

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

Since this driver is controlled through the /sys file system, it requires
administrative privileges to be controlled (ie. to write/read from /sys files).

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

As this is a 3rd-party provided driver, it implies setting up the said
software first. Other than that, only the new driver-specific configuration
values will be added.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <dav.pineau@gmail.com>

Other contributors:
  None

Work Items
----------

Provide a cinder driver leveraging both LVM for non-implemented features and
the driver itself for the natively supported features.

Dependencies
============

None

Testing
=======

The driver does not bring any modification to the existing APIs and behaviors.
As such, no new tempest tests should be required in our understanding.

As this driver relies on a vendor-specific software, the gate obviously cannot
test the driver. We are currently setting up third-party CI testing for both
our previous driver (Scality Sofs) and this new one.


Documentation Impact
====================

None

References
==========

None

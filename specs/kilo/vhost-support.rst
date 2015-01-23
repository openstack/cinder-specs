..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add vHost Executor
==========================================

https://blueprints.launchpad.net/cinder/+spec/vhost-support

The vHost driver was added in the 3.6 Linux kernel. The Linux-IO Target vHost
fabric module implements I/O processing based on the Linux virtio mechanism. It
provides virtually bare-metal local storage performance for KVM guests.
Currently Linux guest VMs are supported.

Problem description
===================

The vHost driver is not a self-contained virtio device, as it depends on
userspace to handle the control plane while the data plane is done in kernel.
This means the data plane does not go through emulations, which can slow down
I/O performance. Cinder today does not provide an option for taking advantage
of the Linux vHost driver.

Use Cases
=========

Proposed change
===============

Add an additional brick executor that knows how to work with vHost.  The
executor will have a first pass implementation of creating, deleting, listing
vHost endpoints through LIO.

Creating the endpoint requires a block device to be available on the machine
that is creating the vHost target. The vHost executor would pass the block
device path to to rtstools, and rtstools will create vHost endpoint with a lun
to the block device.

Alternatives
------------

n/a

Data model impact
-----------------

n/a

REST API impact
---------------

n/a

Security impact
---------------

n/a

Notifications impact
--------------------

n/a

Other end user impact
---------------------

n/a

Performance Impact
------------------

Cinder itself being the control plane will not experience any different
performance. The data plane should experience a greater deal of performance
[1].

Other deployer impact
---------------------

n/a

Developer impact
----------------

n/a


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    thingee

Work Items
----------

* Add vHost executor to brick

Dependencies
============

n/a

Testing
=======

There will be appropriate unit tests available making sure target creation,
deletion, listing works.

Documentation Impact
====================

n/a

References
==========

[1] - http://linux-iscsi.org/wiki/VHost#Linux_performance

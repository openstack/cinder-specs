..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Cinder volume driver for Huawei SDSHypervisor
==============================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/huawei-sdshypervisor-driver

This proposal is to add Huawei SDSHypervisor driver to cinder.

Huawei SDSHypervisor is a storage virtualization solution, is a software
running in host, just focus on the data plane, the goal is to facilitate
the reuse of customer's existing old and low-end devices. SDSHypervisor does
not interact with the storage device in control plane, such as create volume,
create snapshot, do not need third-party device manufacturers to develop any
driver. Administrator just attaches Lun to the hypervisor node as hypervisor
storage entity and hypervisor will provide virtual volume based user's QoS
and patch advance feature to these Lun such as snapshot, link clone, cache,
thin provision and so on.

The purpose of this blue print is to add Huawei SDSHypervisor Driver to Cinder.

Currently there are many heterogeneous storage devices in user product
environment which is low-level device without advance feature, e.g. snapshot,
clone, cache, thin provision, QoS. So adding the SDShypervisor driver will
bring the following advantages:
* Better I/O performance using SDSHypervisor cache feature when there is
SSD in host.
* Facilitate the reuse of customer's existing old and low-end devices, patch
advance feature to these device via SDSHypervisor, e.g. snapshot, clone,
cache, thin provision, QoS
* Help customer use Openstack build his cloud OS. For Cinder, we can't allow
driver to unlimited expand. For manufacturers, they may not want to spend more
time to develop cinder driver for these obsolete and not be maintained
devices. But in the customer's environment there are such devices which need
to be accessed to openstack, in this situation customers can access these
devices to hypervisor, indirect access to Cinder, use Openstack build his
cloud OS.


Problem description
===================

Currently, user can't access Huawei SDShypervisor by Openstack Cinder.

Use Cases
=========

Proposed change
===============

We will add a new Cinder driver that uses socket API interact with Huawei
SDShypervisor storage. SDShypervisor data panel using private Key value
protocal, so we also add a new connector to realize attach/detach volume.

The following diagram shows the command and data paths.

::

                    +------------------+
                    |                  |
                    |  Cinder +        |
                    |  Cinder Volume   |
                    |                  |
                    |                  |
                    +------------------+
                    |                  |
                    |                  |
                    |                  |
                    |                  |
    +---------------+-------+     +----+------------------+
    |                       |     |                       |
    +                       |     |                       |
    |     SDShypervisor     |     | SDShypervisor Driver  |
    |       connector       |     |                       |
    |                       |     |                       |
    +-----------------------+     +-----------------------+
                       |           |
                       |           |
                       |           |
                      CLI     Socket API
                       |           |
                       |           |
                       |           |
                    +--+-----------+---+
                    |                  |
                    |                  |
                    |  SDShypervisor   |
                    |    storeage      |
                    |                  |
                    +------------------+



Add new driver in /cinder/volume/drivers path, and realize cinder driver
minimum features:
* Volume Create/Delete
* Volume Attach/Detach
* Snapshot Create/Delete
* Create Volume from Snapshot
* Get Volume Stats
* Copy Image to Volume
* Copy Volume to Image
* Clone Volume
* Extend Volume

Add new connector in cinder/brick/initiator path, and realize abstract
connector methods:
* connect_volume
* disconnect_volume

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

User will be able to use Huawei SDSHypervisor with Cinder.

Performance Impact
------------------

None

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
  zhangni <zhangni@huawei.com>

Other contributors:
  None

Work Items
----------

Realize Cinder driver minimum features using socket API.
Realize new connector using CLI.
Add unit test code for Huawei SDShypervisor cinder driver and connector.


Dependencies
============

Because Huawei SDShypervisor data panel using private Key-Value protocal,
we will create a new libvirt volume driver in Nova to realize
attach/detach volume to instance. Nova BP page is
https://blueprints.launchpad.net/nova/+spec/huawei-sdshypervisor-volume-driver


Testing
=======

3rd party continuous integration will be done for Huawei SDSHypervisor Driver.


Documentation Impact
====================

The CinderSupportMatrix table and Block storage manual should be updated to
add Huawei SDShypervisor.


References
==========

https://wiki.openstack.org/wiki/Cinder/HuaweiSDSHypervisorDriver

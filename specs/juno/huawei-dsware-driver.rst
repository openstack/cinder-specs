..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Cinder volume driver for Huawei Dsware
==============================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/huawei-dsware-driver

This proposal is to add Huawei Dsware driver to cinder.

Huawei Dsware is a virtual SAN product, virtualizes homogeneous local
disk device from mutiple hosts to provide virtual block device,
while providing advanced features such as snapshot, clone,thin provision,
cache and so on.


Problem description
===================

Currently, user can't access Huawei Dsware by Openstack Cinder.


Proposed change
===============

We will add a new Cinder driver that uses socket API to interact with Huawei
Dsware storage. Dsware data panel using private Key-Value protocal,
so we also add a new connector to realize attach/detach volume.

The following diagram shows the command and data paths.

````

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
    |                       |     |                       |
    |        Dsware         |     |     Dsware Driver     |
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
                    |     Dsware       |
                    |     storage      |
                    |                  |
                    +------------------+


````

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

Add a new connector which can be shared with Huawei SDShypervisor in
cinder/brick/initiator path, and realize abstract connector methods:
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

User will be able to use Huawei Dsware with Cinder.

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
Add CI unit test plugins for Huawei Dsware cinder driver and
connector.


Dependencies
============

Because Dsware data panel using private Key-Value protocal, we will create a
new libvirt volume driver in Nova to realize attach/detach volume to
instance.


Testing
=======

Continuous integration will be done for Huawei Dsware Driver.


Documentation Impact
====================

The CinderSupportMatrix table should be updated to add Huawei Dsware.
https://wiki.openstack.org/wiki/CinderSupportMatrix


References
==========

None

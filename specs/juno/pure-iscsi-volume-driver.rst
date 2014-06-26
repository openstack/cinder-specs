..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Cinder volume driver for Pure Storage FlashArray
================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/pure-iscsi-volume-driver

The purpose of this spec is to add a Cinder volume driver for Pure Storage
FlashArray. It will support the iSCSI protocol and will include at least the
minimum set of features required by the Juno release. We will also set up
automated CI testing on an OpenStack cluster maintained by Pure Storage in
accordance with the third party CI requirement policy.

Pure Storage builds all-flash storage arrays. The active/active controller
architecture is based on the Purity Operating Environment and providing an
adaptive metadata fabric that is scalable, granular to 512B, and protected.


Problem description
===================

Currently, the Pure Storage FlashArray cannot be used a block storage backend
in an OpenStack environment.


Proposed change
===============

Add a Pure Storage FlashArray Cinder driver to the OpenStack Cinder package.

The Pure Storage driver will leverage a thin Python REST client library.
The client library interacts with the REST service hosted on the FlashArray
to perform array management and satisfy Cinder driver API requirements.

::

     Control Flow

                        +------------------+
                        |     Cinder +     |
                        |   Cinder Volume  |
                        +--------+---------+
                                 |
                                 v
                        +------------------+
                        |    Pure Driver   |
                        |                  |
                        +--------+---------+
                                 |
                                 v
                        +------------------+
                        | Pure REST Client |
                        |                  |
                        +--------+---------+
                                 |
                                 v
                        +------------------+
                        | Pure REST Server |
                        |   (FlashArray)   |
                        +------------------+

The Cinder driver will make use of REST client APIs to support:

* Volume Create/Delete
* Volume Attach/Detach
* Snapshot Create/Delete
* Create Volume from Snapshot
* Get Volume Stats
* Copy Image to Volume
* Copy Volume to Image
* Clone Volume
* Extend Volume

In addition to the delivery of the driver implementation, the proposed change
includes the addition of unit tests for the driver. Pure Storage will create
dedicated Jenkins jobs to support CI and Tempest test execution on a Devstack
virtual machine to continuously prove functionality of the driver.


Alternatives
------------

None.


Data model impact
-----------------

None.


REST API impact
---------------

None.


Security impact
---------------

The only security related aspect is that an API token is required by the
driver to leverage the REST client and authenticate against the REST API.

The API token must be specified in the cinder.conf configuration file.


Notifications impact
--------------------

None.


Other end user impact
---------------------

The impact of this change is that OpenStack Cinder will have support for
using a Pure Storage FlashArray as a backing block storage device.


Performance Impact
------------------

None.


Other deployer impact
---------------------

The configuration needed to leverage the Pure Storage driver includes:

* volume_driver - Specifies the Pure Storage FlashArray driver module.
* pure_target - The address of the FlashArray storage target.
* pure_api_token - An API token created on the FlashArray to authenticate
  REST clients, which the driver will make use of.

And optionally:

* pure_host_name - The name of a host object on the FlashArray to associate
  with IQNs and volume connections. This defaults to the name "OpenStack" if
  not provided.


Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  victor-ying <victor.ying@purestorage.com>

Other contributors:
  wes-w <wes@purestorage.com>
  zach-olstein <zach.olstein@purestorage.com>


Work Items
----------

* Complete Cinder Pure Storage driver implementation.
* Complete Cinder Pure Storage driver unit tests.
* Pass automated Tempest tests to prove functionality.
* Integrate Devstack VM and Jenkins job into Pure Storage Jenkins system.


Dependencies
============

The driver has a dependency on a REST client library provided by Pure Storage
that allows developers to easily build Python applications built on REST API
functionality.

The library is currently not available as a pip-installable Python package,
so the library module will be committed along with the driver implementation.


Testing
=======

As mentioned in work items, the delivery of the driver will be accompanied
by a suite of unit tests for the driver (that use Mock to isolate driver
code from the FlashArray). Additionally, continuous integration through
Jenkins and automated Tempest test runs are required for the driver to be
accepted.


Documentation Impact
====================

Pure Storage should be listed as having a supported Cinder driver on the
CinderSupportMatrix:
https://wiki.openstack.org/wiki/CinderSupportMatrix


References
==========

None.

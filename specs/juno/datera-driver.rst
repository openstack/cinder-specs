..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Datera Driver
==========================================

https://blueprints.launchpad.net/cinder/+spec/datera-driver

Datera storage driver in Cinder.

Problem description
===================

Integration for Datera storage is not available in OpenStack.

Use Cases
=========

Proposed change
===============

Add a Cinder driver that can allow OpenStack services communicate with Datera
storage for both vHost and ISCSI.

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

n/a

Other deployer impact
---------------------

The deployer needs to set the cinder.conf to the right `volume_driver`.

    volume_driver=cinder.volume.drivers.datera.DateraDriver

ISCSI would require setting up `san_ip`, `san_login` and `san_password`
appropriately.

vHost would have dependencies on using Linux-IO with the vHost fabric module.
The target_helper in the cinder.conf needs to be set to lio_vhost.


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

* Write driver with ISCSI support.
* Write unit tests for ISCSI support.
* Provide cert tests with ISCSI support.
* Write driver with vhost support.
* Write unit tests for vhost support.
* Provide cert tests with vhost support.
* Provide CI with ISCSI and/or vhost support.

Dependencies
============

* Need vHost connector [1].

Testing
=======

* Unit tests
* CI testing

Documentation Impact
====================

n/a

References
==========

[1] - https://review.openstack.org/103048

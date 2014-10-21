==========================================
IBM FlashSystem Cinder Driver
==========================================

https://blueprints.launchpad.net/cinder/+spec/ibm-flashsystem-driver

This blueprint is to add OpenStack Cinder support for IBM FlashSystem. The 
driver communicates with FlashSystem using SSH and support FC protocols 
at present.

Problem description
===================

User Interface of IBM FlashSystem is based on SVC like IBM Storwize. But it's
a totally unlike storage backend and it's hard to rework the Storwize driver. 
The functionality is completely different.
In the future, it is likely that the two systems will diverge in their similarity.
The development felt it was best to create a separate driver.

Proposed change
===============

The new driver will add support for IBM FlashSystem as backend storage
in Cinder.

The driver supports the following APIs:
* Volume Create/Delete
* Volume Attach/Detach
* Snapshot Create/Delete
* Create Volume from Snapshot
* Get Volume Stats
* Copy Image to Volume
* Copy Volume to Image
* Clone Volume
* Extend Volume

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

None

Performance Impact
------------------

None

Other deployer impact
---------------------

The driver can be configured with the following parameters in cinder.conf:
* san_ip - IP to SSH management interface on FlashSystem
* san_login - user name for SSH management interface
* san_password - password for user
* san_ssh_port - port for SSH management interface

Currently IBM FlashSystem Cinder driver only works when open_access=off.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Edwin Wang

Primary assignee:
  edwin-wang

Other contributors:
  None

Work Items
----------

Implement SSH management interface for each functionality.

Dependencies
============

None

Testing
=======

Test using existing test infrastructure according to driver submission steps.

Documentation Impact
====================

Support Matrix needs to be updated to include IBM FlashSystem support.
https://wiki.openstack.org/wiki/CinderSupportMatrix

Block storage documentation needs to be updated to include IBM FlashSystem volume 
driver information in the volume drivers section.
http://docs.openstack.org/

References
==========

http://docs.openstack.org/developer/cinder/api/cinder.volume.driver.html?highlight=volume%20driver#module-cinder.volume.driver

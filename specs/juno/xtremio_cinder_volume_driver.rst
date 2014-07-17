..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
XtremIO storage Cinder volume driver
==========================================

https://blueprints.launchpad.net/cinder/+spec/xtremio-cinder-volume-driver

This is to enable OpenStack to work on top of XtremIO storage.

Problem description
===================

This is a new Cinder driver that would enable Open Stack to work on top of
XtremIO storage.
The following diagram shows the command and data paths.

::

 +----------------+                 +--------+---------+
 |                |  Command        |                  |
 |                |  Path           |  Cinder +        |
 |  Nova          +---------------> |  Cinder Volume   |
 |                |                 |                  |
 |                |                 |                  |
 +-----+----------+                 +--------+---------+
       |                                     |
       |                                     |
       |                                     |
       |                                     |
       |                                     |  +------------------+
       |                                     |  |                  |
  Command                                    +--+                  |
  Path +                                        |  XtremIO Driver  |
       |                                        |                  |
       |                                        |                  |
       |                                        +------+-----------+
       |                                               |
       |                                               |
       |                                               +
       |                                        XtremIO Rest API
       |                                               |
       v                                               |
                                                       |
 +----------------+                                    |    +-----------------+
 |                |                                    |    |                 |
 |  Compute       |                                    |    |                 |
 |                |                                    +---->    XtremIO      |
 |                |           Data Link                     |    storeage     |
 |                +-----------------------------------------+                 |
 +----------------+                                         +-----------------+


Proposed change
===============

2 new volume drivers for iSCSI and FC should be developed, bridging Open stack
commands to XtremIO managment system (XMS) using XMS Rest API.
The drivers should support the following Open stack actions:

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

This feature will use native snapshot so the user can expect great speed up to
all snapshot/clone related actions.

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
  shay-halsband

Other contributors:
  None

Work Items
----------

Implement REST API client to XMS
Implement Logic for each functionality to support both iSCSI and FC


Dependencies
============



Testing
=======

Continuous integration as required for all drivers in the Juno timeframe

Documentation Impact
====================

Add documntation on how to install and use the drivers.

References
==========

* http://docs.openstack.org/developer/cinder/api/cinder.volume.driver.html?highlight=volume%20driver#module-cinder.volume.driver
* XtremIO REST API

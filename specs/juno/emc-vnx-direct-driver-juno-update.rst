..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
EMC VNX Direct Driver Update
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/emc-vnx-direct-driver-juno-update

Add more functionalities into EMC VNX Direct Driver.

Problem description
===================

The following new functionalities will be added in this update:

* FC Support
* FC Auto Zoning Support
* Read-only Volume Support
* External Volume Management Suppport
* Advance LUN Features
    * Compression Support
    * Deduplication Support
    * FAST VP Support
    * FAST Cache Support
* Initiator Auto Registration
* Storage Group Auto Deletion
* iSCSI Target Portal Selection
* Multiple Authentication Type Support
* Security File Support
* Storage-Assisted Retype
* Storage-Assisted Volume Migration
* SP Toggle for HA


Proposed change
===============

FC Support
    Add a new driver class.

FC Auto Zoning Support
    Add logic in intialize_connection() and terminate_connection() according
    to blueprint
    https://blueprints.launchpad.net/cinder/+spec/cinder-fc-zone-manager and
    the fix for bug https://bugs.launchpad.net/cinder/+bug/1308318.

Read-only Volume Support
    Add logic in intialize_connection() according to blueprint
    https://blueprints.launchpad.net/cinder/+spec/read-only-volumes.

External Volume Management Suppport
    Implement driver API according to blueprint
    https://blueprints.launchpad.net/cinder/+spec/add-export-import-volumes.

Advance LUN Features
    Add logic to support more extra spec key-value pairs

Initiator Auto Registration
    Do logic in intialize_connection() to do EMC-specific initiator
    registration.

Storage Group Auto Deletion
    Add resource recycling logic in intialize_connection()

iSCSI Target Portal Selection
    Add logic in intialize_connection() to select target portal that is
    pingable from the initiator.

Multiple Authentication Type Support
    Add new option in configuration file so that the driver can use different
    authentication types that have been supported by VNX arrays.

Security File Support
    Add a new option in configuration file so that encrypted credentials can
    be used instead of credentials in plain text.

Storage-Assisted Retype
    Implement driver API according to blueprint
    https://blueprints.launchpad.net/cinder/+spec/volume-retype.

Storage-Assisted Volume Migration
    Implement migrate_volume() so that VNX native LUN Migration is leveraged.

SP Toggle for HA
    Original implementation only send management requests to one SP. Since VNX
    arrays have dual SPs, the driver is enhanced to send requests to the other
    SP is one is down.

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

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jeegn-chen

Other contributors:
  None

Work Items
----------

* Implement driver changes


Dependencies
============

* NaviSecCLI (a.k.a. Navisphere CLI)
    * For Ubuntu x64, DEB is available in
        * EMC OpenStack Github: https://github.com/emc-openstack/naviseccli
    * For all other variants of Linux, Navisphere CLI is available at
        * Downloads for VNX2 Series:
          https://support.emc.com/downloads/36656_VNX2-Series or
        * Downloads for VNX1 Series:
          https://support.emc.com/downloads/12781_VNX1-Series.


Testing
=======

Tempest test will used to qualify the driver update.


Documentation Impact
====================

Need to update EMC VNX Direct Driver section of OpenStack Configuration
Reference.


References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
EMC VMAX Driver Update
==========================================

https://blueprints.launchpad.net/cinder/+spec/emc-vmax-driver-juno-update

This driver is an enhancement of the EMC SMI-S driver for VMAX. In Juno,
the support for VNX will be removed from the SMI-S driver. Moving forward,
this driver will support VMAX only. Some new features will be added for
VMAX.

Problem description
===================

The existing EMC SMI-S iSCSI and FC driver has some missing features
for VMAX.  In previous release, support for Extend Volume and Create
Volume from Snapshot were only implemented for VNX. In Juno, these
features will be added for VMAX.

In previous release, masking view, storage group, and initiator group
need to be created ahead of time. In Juno, this will be automated.

Use Cases
=========

Proposed change
===============

The following features will be added to the SMI-S based driver to support
VMAX:

* Extend volume
* Create volume from snapshot
* Dynamically creating masking views, storage groups, and initiator groups
* Striped volumes
* FAST policies

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

User will be able to use the new features.  The feature that dynamically
creates masking views, storage groups, and initiator groups will greatly
improve user experience.

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
  xing-yang

Other contributors:
  None

Work Items
----------

* Extend volume
* Create volume from snapshot
* Create masking views, storage groups, and initiator groups dynamically
* Striped volumes
* FAST policies

Dependencies
============

None

Testing
=======

New features need to be tested.

Documentation Impact
====================

Need to document the changes in the block storage manual.

References
==========

None

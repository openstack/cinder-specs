..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Support iSER driver within the ISCSIDriver flow
===============================================

https://blueprints.launchpad.net/cinder/+spec/support-iscsi-driver

This purpose of this BP is to avoid code duplications of classes and parameters,
and to prevent instability in the iSER driver flow, for the TGT and LIO cases.

Problem description
===================

#. Currently the iSER driver is supported over TGT only, without LIO support.
#. There are a couple of iSER classes that inherit from iSCSI driver/target
   classes, but most of their functionality is the same as the iSCSI classes.
   This code duplication causes instability in the iSER driver code, when new
   features or changes are added to the iSCSI driver flow.

Proposed change
===============

These two problems can be solved by adding a small fix, which includes a new
enable_iser parameter within iSCSI Tgt/LIO classes.

All that is needed for RDMA support over iSER, in the Tgt and LIO cases, is
to set just one extra parameter in the volume creation stage.

A deprecation alert will be added to ISERTgtAdm, since This change will act as
a replacement to the current iSER Tgt code.

The Nova part of this spec is specified at:
https://review.openstack.org/#/c/130721/

Alternatives
------------

Leaving ISERTgtAdm, LVMISERDriver, ISERDriver and iser_opts the way they are,
or just deprecating a part of them (but it will miss the purpose of this code
refactoring).

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

Adding a new "enable_iser" parameter, set to "False" by default.
This parameter will be used in TGT or LIO volume creation for setting
RDMA based portals.

This single parameter will deprecate all iser_opts parameters, that are
a duplication from iscsi parameters.

Developer impact
----------------

This change will simplify the maintanance of the iSER driver flow, since no
extra classes or duplicated parameters will be used.

Implementation
==============

Assignee(s)
-----------

Aviram Bar-Haim <aviramb@mellanox.com>

Work Items
----------

#. Fix bug https://bugs.launchpad.net/cinder/+bug/1396265 and add the correct
   driver parameter, with a configurable value to VOLUME_CONF and
   VOLUME_CONF_WITH_CHAP_AUTH.
#. Add a new "enable_iser" parameter that is set to false by default.
#. Set the driver parameter at the VOLUME_CONFs template in TgtAdm for the
   TGT case.
#. Add _set_iser(1) on the network portal object in rtslib for the LIO case,
   according to "enable_iser" value.
#. Set ISCSIDriver's "driver_volume_type" to "iscsi" or "iser" value, according
   to the "enable_iser" value.

Dependencies
============

None

Testing
=======

HW that supports RDMA is required in order to test volume attachment over
iSER.

A new unit test will be added with the new enable_iser parameter over
iSCSI volume driver.

Documentation Impact
====================

After adding the new enable_iser parameter, An updated iSER configuration
guidelines will be added to:

* https://wiki.openstack.org/wiki/Mellanox-Cinder
* http://docs.openstack.org/juno/config-reference/content/lvm-volume-driver.html
* http://community.mellanox.com/docs/DOC-1462

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
 Add support for chiscsi iscsi helper
=====================================

https://blueprints.launchpad.net/cinder/+spec/chiscsi-iscsi-helper

The Chelsio iSCSI target(chiscsi) serves as a drop in replacement for the IET
target, aiming to provide the same functionality as IET. This spec aims at
adding support for said target implementation as a pure iscsi_helper.

chiscsi supports offloading of iSCSI PDU's when required hardware (supported
Chelsio Network cards) is available but will work on any regular NIC as well.
Offloading or lack thereof requires no user intervention once target drivers
are installed. Implementation is initiator agnostic as well, no changes needed
on initiator side.

Problem description
===================

chiscsi target is not currently supported by openstack

* For a Deployer trying to use offloaded iSCSI support on target side, no
  option is currently available.
* Manual intervention is currently required to export volumes, as cinder does
  not understand chiscsi target implementation.

Use Cases
=========

Proposed change
===============

Add one more iscsi_helper option to cover chiscsi, the driver for this will
interact with the chiscsi target implementation to provide same functionality
as iet.

Use of offloading is dependent on required hardware being present but is
completely optional. No intervention is required to enable offload and
offloading will happen in a manner completely transparent to initiator side.

No initiator side changes are required to make use of chiscsi, with or without
offload support. No extra configuration options are required.

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

If iSCSI offload is available, there is a significant performance boost to be
gained. If offloading is not used, performance and resource usage would be
roughly on par with IET or better.

Other deployer impact
---------------------

* No new config options are required besides an extra allowed value for
  'iscsi_helper' that would need to be explicitly set to 'chiscsi'.
* chiscsi target needs to be installed before it can be used.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  anish7

Other contributors:
  kxie

Work Items
----------

Using iet helper as base, create iscsi_helper for chiscsi, with equivalent
commands for all required apis


Dependencies
============

* Ability to use chiscsi target obviously depends on target driver being
  installed, and command utility available on path. No other dependencies

Testing
=======

Current test for IET target should work just fine for chiscsi

Documentation Impact
====================

None except listing chiscsi as an available iscsi_helper

References
==========

* http://www.chelsio.com/iscsi-target-software/

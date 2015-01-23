..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
NetApp Data ONTAP FibreChannel drivers
======================================

https://blueprints.launchpad.net/cinder/+spec/add-fibre-channel-support-to-
netapp-drivers

NetApp's Cinder volume drivers for Data ONTAP support iSCSI, and we are adding
FibreChannel support as well.

Problem description
===================

Cinder volume drivers must extend the protocol-specific subclass of
VolumeDriver (i.e. ISCSIDriver or FibreChannelDriver).  The inheritance model
employed by the NetApp iSCSI drivers is not conducive to adding FibreChannel
support to their extant iSCSI capability.

Other vendors have moved common, protocol-agnostic driver code into library
modules and called those from the actual driver modules, which then become
very thin indirection layers.  NetApp will take this approach as well.

Use Cases
=========

Proposed change
===============

We will make the following changes:

* Move all iSCSI Data ONTAP driver code into library modules (base class plus
  subclasses for 7-mode and cluster-mode specializations).

* Convert the driver modules for iSCSI (7-mode and cluster-mode) into thin
  indirection layers.

* Add corresponding FibreChannel driver modules that call (mostly) the same
  logic in the library modules.

* Enhance the libraries' handling of igroups and initiators to be multi-
  initiator-aware, since FC HBAs typically report multiple host-side WWPNs to
  Nova/Cinder.

* Enhance the library modules to handle FC-specific attach/detach operations
  in the same way as other FC drivers.  We will take full advantage of the
  Zone Manager code in Cinder.

* Enhance our unit tests to cover all the proposed changes, and continue the
  multi-release process of moving NetApp's Cinder unit tests into the correct
  directory structure.

The refactoring needed in the iSCSI drivers to pave the way for FC drivers
will be submitted first, followed by a separate submission to add the FC
drivers themselves.

Alternatives
------------

In addition to the library approach, we also considered building our drivers
using Python mixins, which would eliminate the indirection layer in the driver
modules.  The problem with mixins is that IDEs cannot follow references, and
the mixins cannot be standalone modules since they need common data such as
Cinder configuration objects.  We prototyped both approaches, but the library
solution is much more convenient to work with.

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

One new NetApp-specific config flag is required: netapp_partner_backend_name

NetApp Data ONTAP (7-mode) storage controllers typically operate in HA pairs,
and in a FibreChannel (FC) environment each controller presents LUNs from its
partner into the FC fabric(s).  So the driver must be able to read the FC
port info and LUN map info from not only the controller owning a LUN but also
from its HA partner.  The added flag identifies the config file stanza for the
HA partner (which is always its own Cinder backend).  This approach is much
cleaner than duplicating all the partner connection details in each backend
stanza.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Clinton Knight (cknight)

Other contributors:
  Rushil Chugh (rushil)


Work Items
----------

* Set up FC dev/test environment.

* Refactor drivers to move block storage code into reusable libraries.

* Create FC driver modules for Data ONTAP (7-mode & cluster-mode).

* Update & extend unit tests accordingly.


Dependencies
============

* The refactoring to prepare for FC drivers was captured in a separate
  blueprint: https://blueprints.launchpad.net/cinder/+spec/netapp-cinder-
  driver-refactoring-phase-1


Testing
=======

Running our existing tempest tests in the FC dev/test environment is the
initial step to know the FC-specific changes in the driver libraries are
working.  Regression tests in iSCSI environments will ensure nothing was
broken.

We are also building a CI system, which will be extended during Kilo to
include running tempest in an FC environment.


Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

* The FC implementation described above was informed by pre-existing FC
  drivers from other vendors, most notably the HP/3PAR one.  A conversation
  with Walter Boring was very helpful.


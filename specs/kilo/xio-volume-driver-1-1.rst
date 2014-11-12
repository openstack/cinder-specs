..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================================
Add QoS, media affinity and thin allocation support to X-IO volume driver
==========================================================================

https://blueprints.launchpad.net/cinder/+spec/xio-volume-driver-1-1

This blueprint covers enhancements of X-IO volume driver to add support for:
- QoS specs - add ability to pass down IOPSmin, IOPSmax and IOPSburst to 
ISE storage system.
- Media affinity - add ability to pass down requested media type for volume.
Media types supported: Flash, CADP (hybrid) and HDD only.
- Add support for volume retype.
- Thin allocation support.

Problem description
===================

QoS support:
Allow the end user to specify IOPSmin, IOPSmax and IOPSburst for a volume.

Affinity support:
Allow the end user to specify media type for volume.

Volume retype support:
Allow the end user to change the volume type for an existing volume.
The volume will be updated to align with the new type accordingly.

Thin allocation support:
Allow the end user to specify that the volume should be thinly allocated.

Proposed change
===============

Extra specs and QoS specs for the specified volume type will be parsed and
passed to the ISE storage system as part of the REST call to create volume.

Retype will use modify volume REST command to change volume attributes on ISE
volume.

Extra-specs:
Affinity:Type - specifies media type. Valid options: flash, cadp, hdd.

Example: To create volume type for flash media type.

cinder type-create flash Affinity:Type=flash

QoS-specs:
QoS:IOPSmax - specifies max IOPS.
QoS:IOPSmin - specifies min IOPS.
QoS:IOPSburst - specifies burst IOPS.

Alloc:Type - specifies allocation type. Valid options: thin, thick.

Alternatives
------------

None

Data model impact
-----------------

None. Changes are local to X-IO volume driver.

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

None. Existing cinderclient commands are used to create volume-types
that specifies QoS, affinity.

Performance Impact
------------------

None

Other deployer impact
---------------------

None. See above for new spec options supported.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  richard-hedlind

Other contributors:
  None

Work Items
----------

Affinity support:
Rework driver to support extra-specs in create_volume API.
Affinity attribute will be copied from source volume in case of clone.

Implementation complete.

QoS support:
Rework driver to support qos-specs in create_volume API. Implemented.
QoS attributes will be copied from source volume in case of clone.

Implementation complete.

Retype:
Add retype API to driver. API uses REST API modify volume to change attributes
for existing volume.

Implementation complete.

Thin allocation:
Pass down thin allocation flag if ISE storage array has support for it.

Implementation in progress.

Additional unit tests:
Add unit tests to test out APIs for affinity, qos, retype, thin.

Implementation in progress.

Dependencies
============

Dependent on approval of base version of X-IO volume driver:

https://blueprints.launchpad.net/cinder/+spec/xio-iscsi-fc-volume-driver

https://review.openstack.org/#/c/116186/

Testing
=======

Test using existing test infrastructure according to driver submission steps.
Tests will be added to test_xio.py to cover affinity, qos, retype and thin.

Documentation Impact
====================

Support Matrix needs to be updated to include X-IO support.
https://wiki.openstack.org/wiki/CinderSupportMatrix

Block storage documentation needs to be updated to include X-IO volume driver
information in the volume drivers section.
http://docs.openstack.org/

References
==========

None

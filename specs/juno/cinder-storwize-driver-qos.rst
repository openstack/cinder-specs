..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Add QoS capability to the IBM Storwize driver
=============================================

https://blueprints.launchpad.net/cinder/+spec/cinder-storwize-driver-qos

Storwize driver can provide the QoS capability with the parameter of
IO throttling, which caps the amount of I/O that is accepted. This
blueprints proposes to add the QoS support for Storwize driver.

Problem description
===================

Storwize backend storage can be enabled with the QoS support by limiting
the amount of I/O for a specific volume. QoS has been implemented for
some cinder storage drivers, and it is feasible for the storwize driver
to add this feature as well.

Proposed change
===============

To enable the QoS for storwize driver, an extra spec with IOThrottling as the
key needs to bind to a volume type. All the following changes apply to the
Storwize driver code.

Create volume:

* If QoS is enabled and IOThrottling is available as the QoS key, set the I/O
  throttling of the volume into the specified IOThrottling value.

Create clone volume:

* If the QoS is set for the original volume, the target volume needs to set
  to the same QoS.

Create snapshot:

* The QoS attributes will be copied to the snapshot.

Create volume from snapshot:

* If QoS is enabled and IOThrottling is available as the QoS key, set the IO
  throttling of the volume into the specified IOThrottling value.

Re-type volume:

* If a volume type is changed, a different IOThrottling value will apply to
  the volume and the I/O of the volume needs to be set to the new IO
  throttling.


Alternatives
------------

The proposed change follows the pattern, in which other drivers implement the
QoS feature.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

The QoS for Storwize driver can be configurable with a configuration option
in cinder.conf. An extra spec with the key of IOThrottling can bind to a
volume type, so that the volumes with this volume type are guaranteed with
an I/O throttling rate. 

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Vincent Hou

Other contributors:
  TBD

Work Items
----------

* Add a configuration option to allow the user the use the QoS for Storwize
  driver.
* Add the QoS enablement check and set the I/O throttling in create volume.
* Add the QoS enablement check and set the I/O throttling in create volume
  from snapshot.
* Set the I/O throttling to the new volume acording to the original volume
  in create clone volume.
* Copy the QoS attributes to the snapshot in create snapshot.
* Change the I/O throttling of the volume if the volume type is changed.

Dependencies
============

None.

Testing
=======

* Unit tests need to be added to test the QoS for Storwize driver.

Documentation Impact
====================

* Add how to configure the QoS for the Storwize driver in the document.

References
==========

* Add QoS capability to the IBM storwize driver
  https://blueprints.launchpad.net/cinder/+spec/cinder-storwize-driver-qos


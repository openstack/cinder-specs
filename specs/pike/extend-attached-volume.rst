..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Extend a volume while volume is attached to an instance
=======================================================

https://blueprints.launchpad.net/cinder/+spec/extend-attached-volume

This spec covers the Cinder changes in the volume extend volume
flow that will allow volume drivers that support extending for
attached volumes to extend the attached volume.

Extending an attached volume requires Nova side changes. A Nova spec
covers the Nova changes necessary to extend a volume that is
attached to one or more instances.

Cinder will interact with Nova via the existing
os-server-external-events extension to notify Nova that the volume
has changed size and react to the change in size.

After Nova has processed the extend size API call,
the instance administrator might need to take action to make use of the
new space associated with the volume. Within the instance the appropriate
actions must be taken for the OS the instance is running to discover
the new size of the volume. The instance administrator can then make
use of the new space in the volume.


Problem description
===================

Cinder requires that a volume be in the available state.
This requires an attached volume to be detached from an instance before
the volume can be extended in size. This means taking the volume offline
from the standpoint of the application using the volume.


Use Cases
=========

An operator wants to increase the size of a volume that is currently
attached to an instance without detaching that volume from the instance.


Proposed change
===============

In the Cinder extend API, allow the volume extend operation to
continue processing when the volume status is 'in‐use' for volume
drivers that can support extending an 'in‐use' volume.
A volume could be extended if the volume status is 'available' or
'in-use' if the volume driver supports in-use extend.
All other volume status values besides 'available' and 'in-use'
would raise an exception.

A volume driver wishing to support in-use extend must add a new
capability (in-use-extend) that indicates the driver supports
extending a volume while that volume is 'in-use'. If a driver does
not have the capability to extend its volumes while they are
'in-use', the API will fail the extend request with a message
indicating that the volume driver does not support extending
a volume that is in the 'in-use' state.

A new policy will be introduced so a deployer can disable the volume
extend operation on attached volume if the deployment does not fully
support the feature. For example, Nova virt driver not supporting
online volume size extension.

If the volume driver capabilities indicate that the driver supports
in-use extend then the volume status will be changed to 'extending'
and then call the driver extend_volume. After changing
the volume status to 'in‐use', notify Nova that the volume has changed size.
Nova will react to the change in size for all the instances
that have the volume attached.

Alternatives
------------

One alternative is to create a new volume and attach that volume
to the instance. Once the new volume is discovered on the instance,
use LVM to extend the space to the application LV. This is the method
being used today to allow more space to be allocated. This is
undesirable in the long run as each time a volume is extended the
extension results in a new volume to be allocated, discovered on
the instance, and added to a volume group, then extending the LV to
include the volume. Operators have expressed concerns about using
this approach as it eventually encounters limitations in the
number of volumes that can be attached to a specific OS. For the
volume controllers that have the ability to grow a volume while it
is in‐use, it is desirable to use that ability.

Data model impact
-----------------

None

REST API impact
---------------

No changes to the API interface itself, as no change is required to the
arguments to the API. The API documentation will change to reflect
that an in-use volume may now be extended, if the volume driver capabilities
allow in-use extend.

Security impact
---------------

None

Notifications impact
--------------------

TBD

Other end user impact
---------------------

End user will be able to extend their volumes without having to
detach them first.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Driver owners whose backend controllers support extending a volume that is
in-use by an instance may want to enable in-use extend.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
* Mathieu Gagné

Other contributors:
* Gerald McBrearty
* Sam Matzek

Work Items
----------

The Cinder extend volume API will check volume driver capability to see if
the driver supports in-use-extend when the status of the volume is in-use.

The extend_volume rpcAPI will need to change the status of a volume back to
'in-use' if the volume is attached to an instance or 'available' if the volume
is not attached to an instance, unless the driver raises an error which
will cause the volume status to be changed 'error-extending'.


Dependencies
============

* Nova spec: Allow an attached volume to be extended [1]_
  Adds the Nova external events support for volume-changed.
  It will call the os-brick extend_volume API to trigger
  the host kernel size information to be updated on
  the host where the volume has been discovered.

Testing
=======

The following scenarios for in-use volumes will need to be covered in
UT and FVT

1) volume driver without in-use-extend capability.
2) volume driver with in-use-extend capability with Nova not supporting
   the volume-changed external event.
3) volume driver with in-use-extend capability with Nova supporting
   the volume-changed external event.

Documentation Impact
====================

The extend documentation will need to be updated to indicate that a
volume that is attached to an instance can be extended if the driver has
the capability 'in-use-extend'.


References
==========

.. [1] http://specs.openstack.org/openstack/nova-specs/specs/pike/approved/nova-support-attached-volume-extend.html

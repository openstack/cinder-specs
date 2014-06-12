..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Support Volume Num Weighter
===========================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/support-volume-num-weighter

Provide a mean to help improve volume-backends' IO balance and volumes' IO
performance.

Problem description
===================
Currently cinder support choosing volume backend according to free_capacity
and allocated_capacity.
Volume Num Weighter is that scheduler could choose volume backend based on
volume number in volume backend, which could provide another mean to help
improve volume-backends' IO balance and volumes' IO performance.

Explain the benifit from volume number weighter by this use case.

Assume we have volume-backend-A with 300G and volume-backend-B with 100G.
Volume-backend-A's IO capabilities is the same volume-backend-B IO
capabilities.
Each volume's IO usage are almost the same.
Use CapacityWeigher as weighter class.

Concrete Use Case:
If we create six 10G volumes, these volumes would placed in volume-backend A.
All the six volume IO stream has been push on volume-backend-A, which would
cause volume-backend-A does much IO scheduling work. At the same time,
volume-backend-B has no volume and its IO capabilites has been wasted.

If we have volume number weighter, scheduler could do proper initial placement
for these volumes----three on volume-backend A, three on volume-backend-B. So
that we can make full use of all volume-backends' IO capabilities to help
improve volume-backends' IO balance and volumes' IO performance.


Proposed change
===============

Implement a volume number weighter:VolumeNumberWeighter.
1. _weigh_object fucntion return volume-backend's non-deleted volume number by
using db api volume_get_all_by_host.
2. Add a new config item volume_num_weight_multiplier and its default value is
-1, which means to spread volume among volume backend according to
volume-backend's non-deleted volume number.

Since VolumeNumberWeighter is mutually exclusive with
CapacityWeigher/AllocatedCapacityWeigher and cinder's
scheduler_default_weighers is CapacityWeigher, we could set
scheduler_default_weighers=VolumeNumberWeighter in
/etc/cinder/cinder.conf and restart cinder-scheduler to make
VolumeNumberWeighter effect.

VolumeNumberWeighter, whichi provides a mean to help improve
volume-backends' IO balance and volumes' IO performance,
could not replace CapacityWeigher/AllocatedCapacityWeigher,
because CapacityWeigher/AllocatedCapacityWeigher could be used to provide
balance of volume-backends' free storage space when user foucs more on free
space balance between volume-bakends.



Alternatives
------------

None.

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
  ling-yun

Work Items
----------

* Implement Volume Number Weighter
* Add weighter option of Volume Number Weighter to OPENSTACK CONFIGURATION
REFERENCE

Dependencies
============
None

Testing
=======
Set up volume-backend-A with 300G and volume-backend-B with 100G.
Create six 10G volumes, the expected result is 3 volumes in
volume-backend A and 3 volumes in volume-backend B.


Documentation Impact
====================

Add weighter option of Volume Number Weighter to OPENSTACK CONFIGURATION
REFERENCE.


References
==========

None

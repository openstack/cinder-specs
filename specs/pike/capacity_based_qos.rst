..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Capacity-based QoS
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/capacity-based-qos

QoS values in Cinder currently are able to be set to static
values.  This work proposes a way to derive QoS limit values
based on volume capacities rather than static values.


Problem description
===================

This proposes a mechanism to provision IOPS on a per-volume basis with
the IOPS values adjusted based on the volume's size.  (IOPS per GB)


Use Cases
=========

A deployer wishes to cap "usage" of this system to limits based
on space usage as well as throughput, in order to bill customers
and not exceed limits of the backend.

Associating IOPS and size allows you to provide tiers such as

 Gold:    1000 GB at 10000 IOPS per GB
 Silver:  1000 GB at 5000 IOPS per GB
 Bronze:   500 GB at 5000 IOPS per GB


Proposed change
===============

Allow creation of qos_keys:
   read_iops_sec_per_gb
   write_iops_sec_per_gb
   total_iops_sec_per_gb

These function the same as our current <x>_iops_sec keys,
except they are scaled by the volume size.


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

New optional qos spec values.

Off by default, opt-in.

Developer impact
----------------

None


Implementation
==============


Assignee(s)
-----------

Primary assignee:
  eharney


Work Items
----------

* https://review.openstack.org/#/c/447127/


Dependencies
============


Testing
=======


Documentation Impact
====================

Document new fields available in qos types.


References
==========

Code: https://review.openstack.org/#/c/447127/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Differentiate thick and thin provisioning logic in scheduler
============================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/differentiate-thick-thin-in-scheduler

Currently in the capacity filter and weigher of the scheduler, we use the
logic to evaluate whether there is enough capacity to thin provision a volume
on a backend if the driver reports `thin_provisioning_support` to be True.
However, a driver may be able to support both thin and non-thin provisioning.
The logic does not check whether the user wants the volume to be provisioned
as thin or not. This blueprint proposes to fix the problem by checking
`thick_provisioning_support` in extra specs of the volume type.

Problem description
===================

If a driver reports both `thin_provisioning_support` to True and
`thick_provisioning_support` to True for a pool that can support both thin and
thick luns, and the user wants to create a thick lun, the logic in the
scheduler would wrongly use the logic for thin provisioning to make decisions.
It would make decisions based on `provisioned_capacity_gb` and
`max_over_subscription_ratio`. This could potentially lead to over
provisioning.

In this spec, we are trying to address this problem by checking whether the
new volume is thin or thick and then use logic accordingly to make decisions.

Use Cases
=========

Currently a driver can report both `thin_provisioning_support` to True
and `thick_provisioning_support` to True if it has a pool that can
support both thin and thick. However the logic in the scheduler
checks capacity for thin provisioning if the driver reports
`thin_provisioning_support` to True even if the volume type specifies
the volume to be thick. The proposed spec wants to fix this problem.

Proposed change
===============

The spec proposes to make the following change in the logic in the capacity
filter and the capacity weigher.

The volume type of the volume to be provisioned will be checked. If
`provisioning_type` is set to `thick` in the extra specs of
the volume type, it will use the thick provisioning logic to evaluate.
Note that this only affects the logic if the driver reports both
`thin_provisioning_support` and `thick_provisioning_support` to True.

Otherwise, the logic remains the same as before.

Alternatives
------------

None.

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

When the admin sets up volume types, he/she needs to set
the following in the extra specs for a thick volume type::

{'thick_provisioning_support': <is> True>} or
{'capabilities:thick_provisioning_support': <is> True>}

Developer impact
----------------

Driver developer should be aware of this extra spec
and handle it accordingly in volume creation.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <xing-yang>

Other contributors:
  <launchpad-id or None>

Work Items
----------

1. Modify capacity filter to check `thick_provisioning_support`.
2. Modify capacity weigher to check `thick_provisioning_support`.

Dependencies
============

None.

Testing
=======

Unit tests will be added for this change.

Documentation Impact
====================

Documentation needs to be changed to include this information.

References
==========

A patch is proposed in Manila to solve a similar problem:
    https://review.openstack.org/#/c/315266/

Note that capabilities reporting for thin and thick provisioning
in Manila is different from that in Cinder. In Manila, a driver reports
`thin_provisioning = [True, False]` if it supports both thin and thick;
In Cinder, a driver reports `thin_provisioning_support = True` and
`thick_provisioning_support = True` if it supports both thin and thick.
Therefore the proposal in this spec is different from the solution in
the Manila patch.

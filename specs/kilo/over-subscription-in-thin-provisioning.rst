..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Over Subscription in Thin Provisioning
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/over-subscription-in-thin-provisioning

This proposal is to introduce a mechanism to allow over subscription in thin
provisioning and also use reserved percentage to prevent over provisioning.

Problem description
===================

There are a couple of problems in capacity reporting in thin provisioning:

* Some drivers are still using 'infinite' and 'unknown' to report capacity.
  This could lead to over provisioning.

* There's no mechanism to allow over subscription in thin provisioning.
  Over subscription allows flexibility in storage allocation and usage and
  it is an important concept in providing thin provisioning support.

Terminologies
-------------

The following terminologies will be used in this spec:

total_capacity: This is an existing parameter already reported by the driver.
It is the total physical capacity.
Example: Assume backend A has a total physical capacity of 100G.

free_capacity: This is an existing parameter already reported by the
driver. It is the real physical capacity available to be used.
Example: Assume backend A has a total physical capacity of 100G.
There are 10G thick luns and 20G thin luns (10G out of the 20G thin luns
are written). In this case, free_capacity = 100 - 10 -10 = 80G.

free: This is calculated in the scheduler by subtracting reserved space
from free_capacity.

volume_size: This is an existing parameter. It is the size of the volume to
be provisioned.

provisioned_capacity: This is a new parameter. It is the apparent allocated
space indicating how much capacity has been provisioned.
Example: User A created 2x10G volumes in Cinder from backend A, and
user B created 3x10G volumes from backend A directly, without using Cinder.
Assume those are all the volumes provisioned on backend A. The total
provisioned_capacity will be 50G and that is what the driver should be
reporting.

allocated_capacity: This is an existing parameter. Cinder uses this to
keep track of how much capacity has been allocated through Cinder.
Example: Using the same example above for provisioned_capacity, the
allocated_capacity will be 20G because that is what has been provisioned
through Cinder. allocated_capacity is documented here to differentiate
from the new parameter provisioned_capacity. If a driver can guarantee
that the backend is used solely for a specific Cinder installation,
then provisioned_capacity could be the same as allocated_capacity.
Only driver has knowledge of this and therefore driver can make a
decision on how to report provisioned_capacity.

max_over_subscription_ratio: This is a new parameter. It is the
float representation of the over subscription ratio when thin provisioning
is involved. It is a ratio of provisioned capacity over total capacity.
Default ratio is 1.0, meaning provisioned capacity cannot exceed the
total capacity. Note that this ratio is per backend or per pool depending
on driver implementation.

reserved_percentage: This is an existing parameter. It represents the
percentage of backend capacity that is reserved. It ranges from 0 to 100.
Default is 0. Note this refers to the real capacity. This is per backend
or per pool depending on driver implementation.

reserved: This is a ratio of reserved_percentage over 100 calculated by the
scheduler.

virtual_free_capacity: This is a parameter calculated based on other
parameters. It refers to how much space a user can still provision apparently.
Note that this is different from free_capacity. free_capacity is
about the real available physical capacity. virtual_free_capacity is
about apparent available virtual capacity.
Example: Assume the total capacity of backend A is 100G. The max over
subscription ratio is 2.0. If no volumes have been provisioned yet,
the virtual_free_capacity is 100 x 2.0 = 200. If 50G volumes have
already been provisioned, the virtual_free_capacity is 200 - 50 = 150.

Use Cases
=========

Proposed change
===============

New parameters in get_volume_stats
----------------------------------
One new configuration options "max_over_subscription_ratio" will be added
to cinder/volume/driver.py. This configuration option will be added to
cinder.conf. It can be configured for each backend when multiple-backend
is enabled.

Note: This configuration option is provided as a reference implementation
and will be used by the LVM driver. However, it is not a requirement for a
driver to use this option from cinder.conf. Driver can choose its own way
if that makes more sense.

In scheduler/host_manager.py, the following additional information sent by
each backend will be saved and will be used later by the scheduler to make
decisions. Note these are reported for a backend or a pool, depending on
driver implementation. In the LVM driver, these will be reported for a
backend (which is treated as one pool).

* provisioned_capacity
* max_over_subscription_ratio

There is a change on how reserved_percentage is used. It was measured
against free capacity in the past. Now it will be measured against
total capacity.

Note: The configuration option max_over_subscription_ratio added in
cinder.conf is for configuring a backend.
For a driver that supports multiple pools per backend, it can report
these ratios for each pool in get_volume_stats.

* Driver can get this option in cinder.conf and report the same ratio
  for the pools that belonging to the same backend.
* Alternatively driver can choose to report different ratio for each pool
  in get_volume_stats, without using this option in cinder.conf.

Capabilities
------------
Driver can report the following capabilities for a backend or a pool:

thin_provisioning_support = True (or False)
thick_provisioning_support = True (or False)

Two capabilities are added here to allow a backend or pool to claim support
for thin provisioning, or thick provisioning, or both.

Volume type extra specs
-----------------------
If volume type is provided as part of the volume creation request, it can
have the following extra specs defined:

'capabilities:thin_provisioning_support': '<is> True' or '<is> False'
'capabilities:thick_provisioning_support': '<is> True' or '<is> False'

Note: 'capabilities' scope key before 'thin_provisioning_support' and
'thick_provisioning_support' is not required. So the following works too:

'thin_provisioning_support': '<is> True' or '<is> False'
'thick_provisioning_support': '<is> True' or '<is> False'

The above extra specs are used by the scheduler to find a backend that
supports thin provisioning, thick provisioning, or both to match the needs
of a specific volume type.

If an extra spec scope key "provisioning:type" is defined, it can be used
by the driver to detemine whether the lun to be provisioned is thin or thick.
The value of this extra spec is either "thin" or "thick". Note this extra
spec is not used by the scheduler to find a backend.

Capacity filter
---------------
In the capacity filter, the following will be evaluated in the decision making
when choosing a backend that fits the criteria:

If (provisioned_capacity + volume_size) / total_capacity >=
max_over_subscription_ratio, the backend will not be chosen to provision
the volume.
Note: This formula will be executed only if "thin_provisioning_support"
is True and max_over_subscription_ratio >= 1.

If ((free_capacity - total_capacity * reserved) * max_over_subscription_ratio)
< volume_size, the backend will not be chosen to provision the volume.
Note: This formula will be executed only if "thin_provisioning_support"
is True and max_over_subscription_ratio >= 1.

If (free_capacity - total_capacity * reserved) < volume_size, the backend will
not be chosen to provision the volume. Note this check was already in the
capacity filter, but the formula is changed to use total_capacity * reserved
instead of free_capacity * reserved.

Capacity weigher
----------------
In the capacity weigher, virtual_free_capacity should be used for ranking
if "thin_provisioning_support" is True. Otherwise, real free_capacity
will be used as before. A change is made to measured reserved space
against the total_capacity.
virtual_free_capacity = total_capacity * max_over_subscription_ratio -
provisioned_capacity - total_capacity * reserved

LVM driver
----------
In the default LVM driver, changes will be made in get_volume_stats which
periodically reports capabilities and the information will be received by the
scheduler.

* Changes will be made in the LVM driver to report provisioned_capacity.
  It makes calls to the LVM class in brick to retrieve volume information
  including capacities.

* The LVM driver will also report max_over_subscription_ratio. This will be
  from the configuration parameters set in cinder.conf.

* While other drivers need to report max_over_subscription_ratio, they are
  not required to read those ratios from cinder.conf.

Changes will also be made in the following LVM driver functions to make sure
over provisioning will not happen even when a request didn't go through the
scheduler:

* create_volume
* extend_volume

The following will be evaluated in the above LVM driver functions:

* If the ratio of the apparent provisioned capacity over real total capacity
  has exceeded the over subscription ratio, the operation will fail.

* If the free space is smaller than the volume size, the operation will fail.

Use cases
---------
The design of this feature will support the following use cases.

Use case 1:
Each volume type has a separate backend or pool. For example, Gold volume
type uses pool gold, Silver volume type uses pool silver, and Bronze
volume type uses pool bronze. Each pool can have a different max over
subscription ratio.

Use case 2:
One volume type is associated with multiple backends or pools. For example,
Silver volume type uses pool 1 and pool 2. Both pools can have the same
max over subscription ratio. Note that capacities for each pool can be
different at any given time.

Use case 3:
One backend or pool is used by multiple volume types. For example, pool 3
is used by volume types Gold, Silver, and Bronze. Assume Gold volume type
uses thick luns only, Silver volume type can have either thick or thin
luns, and Bronze volume type has thin luns only. Because the over subscription
ratio is calculated by the ratio of provisioned_capacity over total_capacity
and all three volume types are sharing the same pool, the ratio will be
the same for all volume types. Gold volume type can guarantee its space
reservation by creating thick luns. The apparent size and the used size
of a Gold volume will always be the same. For a thin lun created as
Silver or Bronze volume type, the apparent size can be bigger than
the real size. Some detailed examples are shown at this etherpad:
https://etherpad.openstack.org/p/cinder-over-subscription-white-board

Alternatives
------------

Without this, we cannot support over subscription in thin provisioning and
there's also no upper limit that prevents over provisioning from happening.

Data model impact
-----------------

N/A

REST API impact
---------------

N/A

Security impact
---------------

N/A

Notifications impact
--------------------

If the capacity usage has exceeded the used ratio or if the provisioned
capacity has exceeded the over subscription ratio, a notification should be
sent. The notification should report the name of the backend or pool and the
capacity information from the backend or pool.  The purpose of the
notification is for the storage administrator to take notice and take actions
to fix the problem.

Notification will also be sent periodically whenever the scheduler receives
an update of capacities from the backend. This will be consumed by Ceilometer.
This was discussed for the Capacity Headroom topic at the summit. The
Ceilometer team will be responsible for the changes required on the Ceilometer
side for this.

Other end user impact
---------------------

There is a new parameter in cinder.conf that end user needs to be aware of.

Performance Impact
------------------

N/A

Other deployer impact
---------------------

New parameters over_subscription_ratio will be added to cinder.conf.

Developer impact
----------------

Drivers should report provisioning capabilities (thin_provisioning_support
and thick_provisioning_support).

Drivers supporting thin provisioning should report provisioned capacity
in addition to free capacity in get_volume_stats.

For drivers supporting thick provisioning only, free capacity will be
used just as before.

For drivers supporting both thin and thick provisioning, provisioned capacity
and free capacity should both be reported.

If there is a range regarding capacity and you are not sure how to report,
please be conservative. For example, if the available capacity is in the
range of 80 to 100 GB, be conservative and report the lower bound 80 GB.

Driver developers can take a look of _update_volume_stats in the LVM driver
as a reference implementation.

Note: This work is also needed for Cinder to use ThinLVM as the default driver
in Kilo.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xing-yang

Other contributors:

Work Items
----------

1. Add max_over_subscription_ratio in driver.py.

2. Modify host_manager.py to update provisioned capacity, over
   subscription ratio by the backends.

3. Modify capacity filter to check whether over subscription ratio
   has been exceeded in a backend.

4. New parameters max_over_subscription_ratio will be added to cinder.conf.

5. LVM driver will be changed to report virtual capacity and over
   subscrption ratio.

6. LVM class in brick will be updated to calculate provisioned capacity.

7. LVM driver functions will be changed to check whether over
   subscription ratio has been exceeded.


Dependencies
============

N/A


Testing
=======

New unit tests will be added to test the changed code.
Testing will be done using the LVM driver for thin provisioning.
Testing will be done to cover the 3 use cases described above.


Documentation Impact
====================

Documentation changes are needed for the following:
New parameter max_over_subscription_ratio will be added to cinder.conf.
Driver needs to add provisioning capabilities (thick_provisioning_suppot,
thin_provisioning_support) and report provisioned_capacity.


References
==========

Examples:
https://etherpad.openstack.org/p/cinder-over-subscription-white-board

Virtual capacity "provisioned_capacity_gb" was discussed in Winston's spec
https://review.openstack.org/#/c/105190/6/specs/juno/volume-statistics-reporting.rst

Kilo design summit session on this topic:
https://etherpad.openstack.org/p/kilo-cinder-over-subscription
https://etherpad.openstack.org/p/kilo-cinder-capacity-headroom

Documentation on the filter scheduler:
http://docs.openstack.org/developer/nova/devref/filter_scheduler.html
Note: This is a document on Nova filter scheduler, but it is very similar to
the Cinder filter scheduler.

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
capacity headroom
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/capacity-headroom

The proposal is to provide a mechanism surfacing the visibility of
the total remaining block storage capacity which is available to
allocate or resize volumes. It will be an indication for deployer
to have the future plan.

Problem description
===================

Currently, there is not a clear way for admin users to know how
much block storage capacity left totally in cinder can be deployed.

* condition with several backends
* condition with a backend supporting "over subscription" which
  temporary capacity not used can be utilized to deploy new volumes
* condition with a backend which reports capacity as "infinite" or
  "unknown"


Use Cases
=========

The proposal calculates the virtual free capacity for each pool
and also sum for each backend. It will notify the info together
with other useful capacity info which already exist to ceilometer
service. The ceilometer service will utilize the notification to
generate capacity samples. The admin users may have a hint about
the trend of cinder capacity usage, and it will be useful for
future capacity planning.


Proposed change
===============

* Calculates the virtual free capacity for each pool which reports
  capcity normally and sums the total for the backend.

  How to calculate the virtual free capacity(virtual_free):

  For thin provisioning, according to the over
  subscription mechanism implemented for LVM,
  the remaining virtual capacity for a pool
  can be used as terminology:
  virtual_free = apparent_available_virtual_capacity.
  It can be calculated by following formula:

  virtual_free = apparent_available_virtual_capacity =
  total_capacity * max_over_subscription_ratio - provisioned_capacity

  For thick provisioning, just use physical capacity.

* Notify the pool capacity and also the total capacity for
  a backend to ceilometer service. The capacity notification
  includes total/free/allocated/provisioned/virtual_free.

  For backend which reports "unknown/infinite", just report
  it as "unknown/infinite".

New methods in HostState
-------------------------------------

* get_capacity()
  - call get_pools()
  - calculate the capacity info for pool and then sum for each backend

Alternatives
------------
Another alternative could be:

* create a new data table which describe capacity info into database
  in scheduler.
* provide a cinder api to retrieve the capacity info from database.

Compare with the proposal, database write operations in scheduler may
cost more.

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

The method update_service_capabilities() in host_manager will
call get_capacity() and send notifications to ceilometer service.
Then ceilometer service can produce capacity samples.

Other end user impact
---------------------

N/A.


Performance Impact
------------------

N/A.

Other deployer impact
---------------------

N/A.

Developer impact
----------------

* It will be better if Cinder volume drivers can report
  provisioned_capacity if it has. This will improve the
  efficiency of the storage utilization.
  Note: This proposal will calculate virtual capacity
  thru provisioned_capacity. Otherwise we will calculate
  physical capacity instead.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  XinXiaohui

Other contributors:

Work Items
----------

* Calculate the capacity:

  - Calculates the total virtual_free capacity for a pool if thin
    provisioning is supported.

    virtual_free = apparent_available_virtual_capacity =
    total_capacity * max_over_subscription_ratio -
    provisioned_capacity

    Otherwise, just use the physical capacity.

* notify capacity (total/free/allocated/provisioned/virtual_free)
  for each pool and also notify total capacity for the backend to
  ceilometer service, if the backend reports "unknown/infinite",
  just report it as it is.

Dependencies
============

* The proposal depends on the over subscription mechanism of backend
  drivers.

Testing
=======

* Unit tests will be added.

Documentation Impact
====================

None.

References
==========

https://etherpad.openstack.org/p/kilo-cinder-over-subscription
https://etherpad.openstack.org/p/kilo-cinder-capacity-headroom

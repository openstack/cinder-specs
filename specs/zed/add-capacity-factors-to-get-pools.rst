..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Add Capacity Factors to scheduler get pools API
===============================================

https://blueprints.launchpad.net/cinder/+spec/add-capacity-factors-to-pool

This spec proposes adding the new capacity factors in the response to the
get-pools API request.

Problem description
===================

Typically there is some monitoring and alerting in place against a cinder
deployment and it's storage array to warn/alert the operator of the cloud
that they are running out of storage, or are out of storage.  The only
mechanism in Cinder now to see the capacity factors of the backends that cinder
is managing is to call the cinder api to get-pools.   This returns some
basic information about the backend/pools.  But it doesn't really tell the whole
story.

The new capacity factors are various aspects of the capacity management that
cinder calculates on each provisioning request.  Each of those factors are
used at runtime to decide if a volume can land on a particular pool.  Those
factors decide if a pool is full or not depending on several factors including,

* thin or thick provisioning support
* total capacity reported by the storage array
* free capacity reported by the storage array
* reserved percentage
* max over subscription ratio

From those configuration settings and pool capabilities a wider set of factors
is calculated to determine the capacity availability on a backend/pool.  All
of those factors determine if a backend/pool has space, is running out of space,
or is out of space for cinder to use.


Use Cases
=========

As an operator of cinder, I want cinder to report all of the factors it uses
and calculates to determine if cinder's backend/pools have space available.
Only cinder has all information and the calculate_capacity_factors() can
create the information the scheduler uses.  It should also report that in
the pools api request, so tooling for monitoring and alerting can be in
sync with the cinder scheduler.

Proposed change
===============

Add the dictionary that is created from calculate_capacity_factors() to each
pool information in the get-pools response.  The capacity factors are not the
same thing as the capabilities being reported now.  The capacity factors are
the breakdown of the capacity calculations based on the driver's reported
capabilities seen by the backend storage.  Depending on the capabilities
in each pool such as thin_provisioning_support and thick_provisioning_support,
factors will be calculated for each thin and thick if the pool supports.

.. code-block:: python

    {
      "total_capacity": 5120.0,
      "free_capacity": 4616,
      "reserved_capacity": 1024,
      "total_reserved_available_capacity": 4096,
      "max_over_subscription_ratio": None,
      "total_available_capacity": 4096,
      "provisioned_capacity": 500,
      "calculated_free_capacity": 3596,
      "virtual_free_capacity": 3596,
      "free_percent": 87.79296875,
      "provisioned_ratio": 0.1220703125,
      "provisioned_type": "thick"
    }

Alternatives
------------

The calculate_capacity_factors() function in utils.py can get copy/pasted into
some external tooling that can do the alerting, but it's subject to getting out
of sync with cinder's version.  Therefore Cinder itself should report all of
those capacity factors for each backend/pool that it calculates.   Then cinder
will be the definitive answer on the capacity it sees and can use.

Data model impact
-----------------

None

REST API impact
---------------

A new microversion would be required and if compatible cinder
would return the capacity_factors dictionary in the pools response.

.. code-block:: console

  GET /v3/{project_id}/scheduler-stats/get_pools?detail=True

.. code-block:: python

  {
      "pools": [
          {
              "name": "pool1",
              "capabilities": {
                  "updated": "2014-10-28T00:00:00-00:00",
                  "total_capacity_gb": 1024,
                  "free_capacity_gb": 100,
                  "volume_backend_name": "pool1",
                  "reserved_percentage": 5,
                  "driver_version": "1.0.0",
                  "storage_protocol": "iSCSI",
                  "QoS_support": false,
                  "thin_provisioning_support": true,
                  "thick_provisioning_support": true,
              },
              "capacity_factors": [
                  {
                      "total_capacity": 1024,
                      "free_capacity": 100,
                      "reserved_capacity": 51,
                      "total_reserved_available_capacity": 973,
                      "max_over_subscription_ratio": None,
                      "total_available_capacity": 973,
                      "provisioned_capacity": 100,
                      "calculated_free_capacity": 873,
                      "virtual_free_capacity": 873,
                      "free_percent": 89.72,
                      "provisioned_ratio": 0.1028,
                      "provisioned_type": "thick"
                  },
                  {
                      "total_capacity": 1024,
                      "free_capacity": 100,
                      "reserved_capacity": 51,
                      "total_reserved_available_capacity": 973,
                      "max_over_subscription_ratio": 2,
                      "total_available_capacity": 1946,
                      "provisioned_capacity": 100,
                      "calculated_free_capacity": 1846,
                      "virtual_free_capacity": 1846,
                      "free_percent": 94.86,
                      "provisioned_ratio": 0.05,
                      "provisioned_type": "thin"
                  }
              ],
          }
      ]
  }


Security impact
---------------

None


Active/Active HA impact
-----------------------

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
  hemna (Walter A. Boring IV)

Work Items
----------

* Add new microversion
* Add ``capacity_factors`` in get-pools API response

Dependencies
============

* This requires the new calculate_capacity_factors() function that is in the
  following review:
  https://review.opendev.org/c/openstack/cinder/+/831247


Testing
=======

Add new unit tests to show the factors being returned in API call.

Documentation Impact
====================

Add documentation to describe the capacity factors and the API response change

References
==========

This was discussed at length at the Zed PTG:
https://etherpad.opendev.org/p/zed-ptg-cinder

Youtube Video of discussion:
https://www.youtube.com/watch?v=6yuOlGckkGE

The capacity factors definitions
https://specs.openstack.org/openstack/cinder-specs/specs/queens/provisioning-improvements.html

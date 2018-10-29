..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Driver reinitialization after failure
==================================================

https://blueprints.launchpad.net/cinder/+spec/driver-initialization-after-fail

This spec proposes support for reintialization of volume drivers after it fails
during startup.

Problem description
===================

When a service starts, at first it does initialization. If something wrong
happens, the service ends.
After initialization completes, it cleans up resources, initializes its driver
and RPC servers. Later the volume service reports its capabilities to scheduler
in fixed interval.

During above progress, some errors lead to the service process exiting,
leaving the volume driver not functioning.

1) Driver not supported.
2) Driver initialization fails.
3) Volume cleanups are processed by parent class CleanableManager.
4) Something wrong with 'publish_service_capabilities' [2].

The driver will not be initialized in above case 1), 2) and 3). As a result,
although the volume service process exists, and tries to publish its service
capabilities to scheduler, but fails every time. This means users have to
restart cinder volume to re-initialize drivers.

When case 4 happens the volume service moves on and can become available if
it succeeds to publish its service to scheduler next time.

Use Cases
=========

When a Cinder volume service starts, sometimes its corresponding storage
services are not ready. But later the storage services become ready. As a
result the volume service can't work properly and can't recover by itself.
But the administrator probably prefer Cinder to automatically recover from
the temporary failures without manual intervention of restarting the service.

Proposed change
===============

The proposal is to

- Reintialize the volume driver when it failed to initialize. This reinitialization
  happens except when the error is unrecoverable. The following lists
  unrecoverable errors:

  1) config error
  2) Driver not supported
  3) lack of python driver library

- Every time when a volume service publishes its capability to scheduler,
  it checks whether the driver is initialized. If it is not initialized
  and can be recovered, it calls init_host[1] to do reinitialization.

For this, the following additional config option would be needed:

- 'reinit_driver_count' (default: 3)
   Set the maximum times to reintialize the driver if volume initialization fails.
   Default number is 3. The value 0 means no limitation.

Alternatives
------------

Compared with listing unrecoverable errors when checking whether retrying, another
way is to keep a list of all recoverable errors and reinitialize on the errors in
the list. The problem is that it tightly depends on exceptions raised by drivers which
may change on different versions.

Data model impact
-----------------

None

REST API impact
---------------

None.

Cinder-client impact
--------------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

Users don't need to restart volume service when the initialization of
drivers fail on recoverable errors.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Lisa Li (xiaoyan.li@intel.com)

Work Items
----------

* Add the option 'reinit_driver_count'.
* Need to differentiate config and library error in drivers, and update these
  exceptions to inherit from same base exceptions. So that we can skip these
  errors when reinitializing.
* Reinitialize volume drivers in 'publish_service_capabilities' [2].
* Add related unit test cases.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

* Add administrator documentation to advertise the option of 'reinit_driver_count'
  for driver reinitialization and explain how this should be used.

References
==========

_`[1]`: https://github.com/openstack/cinder/blob/master/cinder/volume/manager.py#L408
_`[2]`: https://github.com/openstack/cinder/blob/master/cinder/volume/manager.py#L2539

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

During Cinder initialization, for many reasons, the storage backend might not
be ready and responding. In this case, the driver will not be loaded even if
the array becomes available right after.

As there is no retry in Cinder volume service, even later the backend storage
is ready, Cinder volume service can't recover by itself. It needs users
to restart the volume service manually.

Use Cases
=========

When a Cinder volume service starts, sometimes its corresponding storage
services are not ready. But later the storage services become ready. As a
result the volume service can't work properly and can't recover by itself.
But the administrators probably prefer Cinder to automatically recover from
the temporary failures without manual intervention of restarting the service.

Proposed change
===============

The proposal is to

- Allow reinitialization of a volume driver when it failed to initialize.

- Provide a configuration to set the maximum retry numbers.

- The interval of retry will exponentially backoff. Every interval is the
  exponentiation of retry count. The first interval is 1s, second interval
  is 2s, third interval is 4s, and so on.

- Retry will be handled in init_host.

For this, the following additional config option would be needed:

- 'reinit_driver_count' (default: 3)
   Set the maximum times to reintialize the driver if volume initialization fails.
   Default number is 3.

Alternatives
------------

- We also can differentiate whether it should retry with an exception. Like
  import error, config error, it may not retry. But the benefit is not
  very impressive, and implementing the differentiation needs work in every
  driver. As drivers don't differentiate such errors from backend storage
  errors.

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
* Retry to initialize volume drivers when it fails.
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

* None

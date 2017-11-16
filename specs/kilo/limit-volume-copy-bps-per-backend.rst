..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Limit Volume Copy Bandwidth Per Backend
==========================================

https://blueprints.launchpad.net/cinder/+spec/limit-volume-copy-bps-per-backend

Currently, cinder has a global configuration to limit a volume copy bandwidth
to mitigate slow down of instances volume access during large image copies.
( https://blueprints.launchpad.net/cinder/+spec/limit-volume-copy-bandwidth )

This spec proposes to make this config variable per backend.


Problem description
===================

The current global config has some issues.

* When multiple backends exist, we cannot assign different bandwidth limit.
  For instance, it is desirable to allow large bps for SSD than for HDD.

* If volume copy runs concurrently, bandwidth larger than the bps limit is
  consumed. From the viewpoint of QoS (to keep bandwidth for access from
  instances), we should limit total volume copy bandwidth per backend.

Use Cases
=========

Proposed change
===============

Make the cinder.conf parameter 'volume_copy_bps_limit' writable in each
backend section. (If volume_copy_bps_limit is specified in DEFAULT section as
existing design, that value is used as a global default. This keeps backwards
compatibility.)

The bps limit is divided if multiple volume copy operations run concurrently.
For example, if volume_copy_bps_limit is set to 100MB/s, and 2 copies are
running at the same time in a backend, each copy can use up to 50MB/s.
If one of the copies finished, the other copy's limit is updated to 100MB/s.

On initialization of VolumeDriver class, a 'Throttle' class object is created
and stored in '_throttle' member for each backend driver object, that manages
the total bandwidth allowed and volume copy operations in flight.

It should be passed to copy_volume() as following, so that it can set/update
the bandwidth limit.

    volume_utils.copy_volume(..., throttle=self._throttle)

In copy_volume method, Throttle class method is called to setup cgroups
(or some other QoS feature, when the Throttle class is overridden).


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

When volume copy I/O bandwidth is limited, it takes more time to complete
volume copy. Users are required to balance between volume copy performance
and interference to instances performance.


Other deployer impact
---------------------

* This feature is disabled by default. Users who want to use this feature need
  to set 'volume_copy_bps_limit' in cinder.conf.

Developer impact
----------------

* A '_throttle' member is added to VolumeDriver class, which should be passed
  to volume_utils.copy_volume() and related function. Otherwise, bandwidth
  limit will be ignored.

* In Linux environments, '_throttle' object will be implemented using cgroups.
  Drivers for non-Linux environments (e.g. Windows) can override the
  initialization of '_throttle' to use another method for throttling.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tsekiyama (tomoki.sekiyama@hds.com)

Work Items
----------

1. Use 'backend_driver.configuration.volume_copy_bps_limit' etc. if
   'CONF.volume_copy_bps_limit' is not specified.
2. Replace current setup_blkio_cgroup() with new 'Throttle' class
3. Pass the Throttle object from the backend drivers to copy_volume()

Dependencies
============

None

Testing
=======

* With volume_copy_bps_limit = 100MB/s for a fake backend driver:

 * start a volume copy, then the bps limit is set to 100MB/s
 * start a second volume copy, then the limit is updated to 50MB/s for both
 * finish one of the copies, then the limit is resumed to 100MB/s


Documentation Impact
====================

The cinder client documentation about volume_copy_bps_limit in cinder.conf
should address that copy bps limit can be specified for each backend driver
by writing this config in the backend section.


References
==========

None

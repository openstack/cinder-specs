..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Limit Bandwidth of Volume Copy
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/limit-volume-copy-bandwidth

This proposes adding a new config to limit bandwidth for volume copy to
mitigate interference to other instance performance during:

* new volume creation from a image
* backup
* volume deletion (when dd if=/dev/zero etc. is chosen to wipe)


Problem description
===================

Currently, volume copy operations consumes disk I/O bandwidth heavily and may
slow down the other guest instances.

"ionice" option is already implemented in some cases, but it is not always
usable. e.g. When instances directly access to the storage and doesn't go
through I/O scheduler of cinder control node, ionice cannot control I/O
priority and instances access may slow down.


Proposed change
===============

A new config named 'volume_copy_bps_limit' will be added to determine max
bandwidth (byte per second) consumed by volume copy.

When CONF.volume_copy_bps_limit is zero (default), no limitation is applied,
and no cgroup is created.

Otherwise, bandwidth limitation is applied to volume copy. For example, if the
volume copy is done by 'dd' command, it can be implemented by putting 'dd'
into blkio cgroup for throttling.



Alternatives
------------

When volume copy commands have an option for I/O throttling, the usage of such
options are preferable.
Putting whole cinder-volume processes into blkio cgroups could also be a
solution for this, though it is required to provide a way to set rate limit to
newly added block devices when new volume is created.


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

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tomoki-sekiyama-g

Work Items
----------

* Implement cgroup blkio setup functions
* Implement I/O rate limit for volume_utils.copy_volume
* Implement I/O rate limit for other image format such as qcow

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

The cinder client documentation will need to be updated to reflect the new
config.


References
==========

None

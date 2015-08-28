..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Update drivers to new base class structure
==========================================

https://blueprints.launchpad.net/cinder/+spec/abc-driver-update

The new abc structure was introduced by ``bp/abc-volume-drivers`` [1]. All
drivers needs to get updated in order to benefit from the new structure.


Problem description
===================

Instead of raising NotImplementedErrors during runtime this features
allows to discovers the drivers feature set during startup and makes
it discoverable for CI/code check systems.

Use Cases
=========

The support matrix (see [2] for a draft implementation) can be extracted
to see the graduation process of new functionality moving to a common
function implemented by all drivers.

Proposed change
===============

All cinder volume drivers needs to get updated with the following approach::

    class FooDriver(driver.RetypeVD, driver.TransferVD, driver.ExtendVD,
                    driver.CloneableVD, driver.CloneableImageVD,
                    driver.SnapshotVD, driver.BaseVD)

A drivers must inherit from BaseVD and implement the basic functions. In order
to mark that a driver does implement further feature sets it must inherit from
the corresponding class.

If all drivers implement a certain feature set the functions will be moved to
BasicVD at the end.


Alternatives
------------

No porting at all, which would make the [1] pointless.

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

See [1]

Other deployer impact
---------------------

None.

Developer impact
----------------

This change will change all implemented drivers slightly. The functionality
itself shouldn't be changed at all but all driver need to be adopted to the
new class model.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Marc Koderer (m-koderer)

Other contributors:
  All driver maintainers

Work Items
----------

Etherpad if necessary.

Dependencies
============

None.

Testing
=======

Individual driver unit tests needs to get adapted.


Documentation Impact
====================

None.


References
==========

[1]: https://review.openstack.org/#/c/114168/
[2]: https://review.openstack.org/#/c/160346/

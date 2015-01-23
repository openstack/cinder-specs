..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Introduce abstract interface model in volume drivers
====================================================

https://blueprints.launchpad.net/cinder/+spec/abc-volume-drivers

Instead of using a loose interface definition of volumes drivers use the ABC
python library to build abstract classes that enforces the driver to implement
the needed functionality. Goal is to fail fast and not using exceptions during
runtime if the driver is not using the correct interface.


Problem description
===================

The defined volume interface in ``cinder.volume.driver`` is quite lose. A
driver can decide whether to implement a certain functionality or not. From
the outside (manager layer) it is not visible which functionality a driver
implements. So the only way to discover that is to try to call a function of
a feature set and to see if it raises a ``NotImplementedError``.

Use Cases
=========

Proposed change
===============

Build a base VolumeDriver class and subclasses that describe feature sets,
like::

                          +-------------------------+
         +----------------+     BaseVolumeDriver    +---------------+
         |                |       {abstract}        |               |
         |                +-----------^-------------+               |
         |                            |                             |
         |                            |                             |
+--------+-------------+  +-----------+-------------+  +------------+---------+
|  VolumeDriverBackup  |  |   VolumeDriverSnapshot  |  |  VolumeDriverImport  |
|      {abstract}      |  |      {abstract}         |  |      {abstract}      |
+----------------------+  +-------------------------+  +----------------------+


If a driver implements the backup functionality and supports volume import it
should inherit from the interface classes like::

    class FooDriver(VolumeDriverBackup, VolumeDriverImport):
        ...

The management layer can observe the feature set's of a given driver by using
``isinstanceof()``::

    volume_driver = FooDriver(..)
    if isinstanceof(volume_driver, VolumeDriverBackup):
        pass

Usage of python ``ABC`` is preferable since it gives a variety of advantages
(see [1], [2]). It will fail on instantiation level with an TypeExcetion.
Other OpenStack project already using this library (see [3]).

Driver Migration
----------------

Instead of changing all drivers in one big step it better to migrate
stepwise. In order to do so the VolumeDriver class can exist with the same
interface and use ``NotImplementedError`` exceptions as before. With that all
existing drivers can be migrated to the new concept one after another.


Alternatives
------------

- Only implement subclasses and don't use ``ABC``

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

There is no heavy performance implication expected due to the reason ABCMeta
and it's functionality is only very limited used inside of the relevant data
path. The following general object function are potential less performant than
before (see [5])::

- __new__()
- __subclasscheck__(), issubclass()
- __instancecheck__(), isinstance()

The performance is depending on the depths of used class hierarchy. The
proposed concept is quite plain in that aspect (maximum 2 levels). In general
it's a comparison between the used cycles for raising/catching an exception
and iteration in a ``for loop`` over all abstract classes.

Other deployer impact
---------------------

None.

Developer impact
----------------

This change will change all implemented drivers slightly. The functionality
itself shouldn't  be changed at all but all driver need to be adopted to the
new class model.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Marc Koderer (m-koderer)

Other contributors:
  Danny Al-Gaaf (danny-al-gaaf)

Work Items
----------

Will be tracked in etherpad.

Dependencies
============

None.

Testing
=======

Unit tests need to be adapted massively since there are catching
``NotImplementedError`` exceptions all over the place.

Documentation Impact
====================

None.


References
==========

[1]: http://legacy.python.org/dev/peps/pep-3119/
[2]: http://dbader.org/blog/abstract-base-classes-in-python
[3]: http://lists.openstack.org/pipermail/openstack-dev/2013-August/014089.html
[4]: https://bugs.launchpad.net/tempest/+bug/1346797
[5]: https://hg.python.org/cpython/file/2.7/Lib/abc.py

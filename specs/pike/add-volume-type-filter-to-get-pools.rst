..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Support retrieve pools filtered by volume-type
==============================================

https://blueprints.launchpad.net/cinder/+spec/add-volume-type-filter-to-get-pool

Add feature that administrators can get back-end storage pools filtered by
volume-type, cinder will return the pools filtered by volume-type's
extra-specs.

Problem description
===================

Now cinder's ``get-pools`` API doesn't support filtering pools by volume type,
and this is inconvenient when administrators want to know the specified pool's
state which also can meet volume type's requirements, the administrators also
can get all pools and filter them on their own, but it's more complicated and
inefficient. This change intends to cover this situation and bring more
convenience to administrators.

Use Cases
=========

In production environment, administrators often need to have an overall
pool available statistics filtered by volume type, this will help them to make
an adjustment before resources run out.

Proposed change
===============

As we will introduce generalized resource filter in cinder, from the view of
outside cinder, the only thing we should do is to advertise that we can
support this filter now:

From the view of inside cinder. We will mark ``volume-type`` recognizable, and
support for this filter in logic:

* once the volume-type (name or id) is detected, the volume type object will be
  retrieved before scheduler api is called, and will be passed as a filter
  item.

* the ``scheduler.get_pools`` called, which will call the
  ``host_manager.get_pools`` in result.

* the ``host_manager.get_pools`` will collect the pools information as normal,
  and before it returns, the result will be filtered by
  ``host.get_filtered_hosts``.

* the filter properties of ``get_filtered_hosts`` only consists of volume-type
  properties.

* As already proposed by generalized resource filtering [1]_, the changes on
  cinder-client for this feature are not needed.


Alternatives
------------

Administrators also can retrieve and filter on their own, but it's more
complicated and inefficient. This change can reduce the request amount and
filter unnecessary data transmitted from server to client.

Data model impact
-----------------

None

REST API impact
---------------

Get-Pool API will accept new query string parameter volume-type.
Administrators can pass name or ID to retrieve pools filtered.

* ``GET /v3/{tenant_id}/scheduler-stats/get_pools?volume-type=lvm-default``

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

Within generalized resource filtering, we would ultimately have a
``get-pools`` command like this below::

 cinder get-pools --filters volume-type='volume type'

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
  TommyLike(tommylikehu@gmail.com)

Work Items
----------

* Add Get-Pools's filter.
* Add filter logic when retrieving pools.
* Add related tests.


Dependencies
============

Depended on generalized resource filtering [1]_

Testing
=======

1. Unit test to test whether volume-type filter can be correctly applied.
2. Tempest test whether volume-type filter work correctly from API
   perspective.

Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the
   REST API changes.

References
==========

.. [1] https://review.openstack.org/#/c/441516/

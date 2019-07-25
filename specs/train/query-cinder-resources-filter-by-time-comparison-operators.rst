..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================================
Support to query cinder resources filter by time comparison operators
=====================================================================

https://blueprints.launchpad.net/cinder/+spec/support-to-query-cinder-resources-filter-by-time-comparison-operators

Support users can query resources by specifying the time comparison
operators along with created_at or updated_at, and cinder will return all
which matches the time condition.

Problem description
===================

Cinder (also other projects, like heat and neutron) API only support filtering
resources by discrete values, even though cinder resources have timestamp
fields, users can only query resources operated at given time,
not during given period. Users may be interested in resources operated in a
specific period for monitoring or statistics purpose but currently they have to
retrieve and filter the resources by themselves.
This change can bring facility to users and also improve the efficiency of
timestamp based query.

Use Cases
=========

In large scale environment, lots of resources were created in system,
for tracing the change of resource, user or manage system only need to get
those resources which was creatded or changed from some time point, instead of
querying all resources every time to see which was changed.


Proposed change
===============

* Introduce six time comparison operators along with created_at or updated_at
  fields for retrieving resources more flexible. User needs to specify the
  operator first, a colon (:) as a separator, and then the time.
  One step closer, we will also support multi-operators querying at once, a
  comma (,) as the separator.
* Operator 'gt': Return results more recent than the specified time.
* Operator 'gte': Return any results matching the specified time and also any
  more recent results.
* Operator 'eq': Return any results matching the specified time exactly.
* Operator 'neq': Return any results that do not match the specified time.
* Operator 'lt': Return results older than the specified time.
* Operator 'lte': Return any results matching the specified time and also
  any older results.


Alternatives
------------

As discussed in 'Problem description' section, user can retrieve and then
filter resources by themselves, but this approach is neither convenient nor
efficient. Leaving filtering work to database can utilize the optimization
of database engine and also reduce data transmitted from server to client.

Data model impact
-----------------

None

REST API impact
---------------

List API will accept new query string parameters. User can pass the operator
and time to the list API url to retrieve resources created or updated since or
prior to a specific time.
This changes also need to bump the microversion of API to keep forward
compatibility.

* GET /v3/{project_id}/volumes/{detail}?updated_at=gt:2016-01-01T01:00:00,lt:2016-12-01T01:00:00

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

Python client may add help to inform users this new filter. Python client
support dynamic assigning search fields so it is easy for it to
support this new filter.

Performance Impact
------------------

As discussed in 'Alternatives' section, performance can be improved for
timestamp based query by utilizing database engine. Additional, it also can add
index to improve querying performance.

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
  wanghao<sxmatch1986@gmail.com>


Work Items
----------

* Add API filter
* Add querying support in sql
* Add related test


Dependencies
============

None


Testing
=======

1. Unit test to test if those filters can be correctly applied.
2. Tempest test if change filter work correctly from API perspective.

Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.

References
==========

[1]: https://developer.openstack.org/api-ref/image/v2/?expanded=list-images-detail#v2-comparison-ops

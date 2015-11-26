..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Support to query cinder resources filter by change_since
========================================================

https://blueprints.launchpad.net/cinder/+spec/supprot-to-query-cinder-resources-filter-by-change-since

Support users can query resources by specifying the time that resources
are changed since, and cinder will return the all which matches condition.

Problem description
===================

Cinder(also other projects, like heat and neutron) API only supports filtering
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
those resources which was changed from some time point, instead of querying
all resources every time to see which was changed.


Proposed change
===============

* Introduce a new change_since filter for retrieving resources. It
  accepts a timestamp and projects will return resources whose update_at fields
  are later than this timestamp.


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

List API will accept new query string parameter change_since. Users can pass
time to the list API url to retrieve resources operated since a specific time.

* GET /v3/{project_id}/volumes/{detail}?change_since=2016-01-01T01:00:00

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
  wanghao<wanghao749@huawei.com>


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

1. Unit test to test if change_since filter can be correctly applied.
2. Tempest test if change filter work correctly from API perspective.

Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.

References
==========

None

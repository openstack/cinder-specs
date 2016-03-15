..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add Pagination To Other Resources
==========================================

https://blueprints.launchpad.net/cinder/+spec/add-pagination-to-other-resource

This spec aims to continue the work that we did in Liberty, adding pagination
support to other cinder resources.


Problem description
===================

In Liberty release, we have added pagination to backups and snapshots
according bp[1]. There are still some work that hasn't been done yet.
In Mitaka, we intend to add pagination support to CG, Volume Type and
Qos specs.

Use Cases
=========

In large scale cloud systems, end users and manage system that's on top of
Cinder could make quick querying by using pagination, filter and sort
functions to improve performance of querying.

Proposed change
===============

* Consistency Group: Refactor current implementation, using DB pagination
  querying, and add support to filter and sort in querying request.
* Volume Type: Add pagination and sort support in querying request.
* Qos Specs: Add pagination, filter and sort in querying request.
* Add sql pagination querying support as we did in backup and snapshot.

Alternatives
------------

None

Data model impact
-----------------

None


REST API impact
---------------

According the API-wg guideline about pagination, filter and sort[2]::

  GET /v2/{project_id}/{resource}?limit=xxx&marker=xxx&sort=xxx&{filter}=xxx
  RESP BODY: {"resource_links": [{xxx}],
              "resource": [{xxx}, {xxx}, ..., {xxx}]
             }


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
  wanghao<wanghao749@huawei.com>

Other contributors:
  None

Work Items
----------

* Add pagination support to three resources.
  * Implement code in db pagination query.
  * Implement code in list querying api.
  * Test code.
* Update cinderclient to support this functionality for those resources.
* Add change to API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests are needed to be created to cover the code change.


Documentation Impact
====================

The cinder API documention will need to be updated to reflect the REST
API changes.


References
==========
[1]https://blueprints.launchpad.net/cinder/+spec/extend-limit-implementations
[2]https://github.com/openstack/api-wg/blob/master/guidelines/pagination_filter_sort.rst

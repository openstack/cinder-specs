..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Show resource's total count info in list APIs
=============================================

https://blueprints.launchpad.net/cinder/+spec/add-amount-info-in-list-api


This blueprint proposes adding total count info in Cinder's list APIs.

Problem description
===================

Show how many resources a user or tenant has is usually required in the web
portal's summary section, but since we introduced the pagination mechanism,
we only could get the ``{resource}_link`` which tells us you have more
resources. In order to show this kind of total number to the end user, many
clouds have to collect all of the resources while they only have to display
the first page of the resource.

Use Cases
=========

The main use case is to have the administrators or users to know how many
resources they have in total without retrieving and accumulating them all.

Proposed change
===============

Having the total count information in list API response is not only a
requirement for Cinder or OpenStack, it's a common requirement for the
REST APIs, so find out what are others are doing may help us to have a
better API model for this case.

1. Facebook API support the ``total_count`` attribute in their list
   APIs `[1]`_. And this is their API response::

    {
      "data": [],
      "paging": {},
      "summary": {
        "total_count": 100
      }
    }

2. JSON API org adds an example to demonstrate the usage of ``total-pages`` or
   ``count`` in their recommended examples `[2]`_::

    {
        "meta": {
            "total-pages": 10,
            "count": 100
        },
        "data": [],
        "links": {
            "self": "",
            "prev": "",
            "next": ""
        }
    }

3. StackExchange API also adds the total attribute in their
   ``Common Wrapper Object`` `[3]`_. And this is how their response
   looks like::

    {
       "items": [],
       "page": 12,
       "page_size": 100,
       "total": 1200,
    }

Since we have already added the ``{resources}_links`` attribute into our list
API response. It makes more reasonable to have the amount info added into the
response (take volume for example)::

    {
      "volumes_links": [],
      "volumes": [],
      "count": 100
    }

So, this bp proposes to add new attribute ``count`` in our list APIs (
including index and detail).

Based on our current pagination system `[4]`_, if we add the ``count``
attribute into our response body in default, the db's query statement would
be executed twice for only one query which obviously has a performance
impact, considering not every request requires this kind of info, the
additional query parameter is required to turn this on when listing
resource::

    /v3/{tenant_id}/{resource}?with_count=true
    /v3/{tenant_id}/{resource}/detail?with_count=true

Also, this is recommended by OpenStack API-WG `[5]`_.

For the first step we only plan to add the summary support for our main
resources: ``volumes``, ``snapshots`` and ``backups``.

Alternatives
------------

There are few alternative solutions for this requirement, let's list and
compare them all here.

1. Add amount information in response header::

    X-resource-count: 200

The disadvantage of this is it will add some burden to the API customers,
we don't have any history for adding useful content in response
headers. Also it makes more difficult for documentation.

2. Add explicit API for each resources::

    POST: /v3/{tenant_id}/{resources}/action {os-count}

This change involves a lot of modifications and creates several new APIs.
Adding this amount of APIs for such a simple API change is overdesign.

3. Create a new API for different resources::

    POST: /V3/{tenant_id}/action {os-count}
    BODY:
        {
            "resource": "volume",
            "user": "user_1",
            "other_filter": "other_value"
        }

For this change, one more API request is required if the end user wants to
know how many resources in total when listing resources.


Data model impact
-----------------

None

REST API impact
---------------

Microversion bump is required for this change.

Cinder-client impact
--------------------

All of the list commands will be upgraded to support this.

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

Since we will add additional ``COUNT()`` statement if the list
APIs are requested with the summary option, there would be negative
performance impact on those APIs, especially when there are a lot
of data in database.

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
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Add summary option support in list APIs
* Add related unit testcases
* Update cinder-client and OSC.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

Update API documentation.

References
==========

_`[1]`: https://developers.facebook.com/docs/graph-api/reference/v2.1/user/friends
_`[2]`: http://jsonapi.org/examples/
_`[3]`: https://api.stackexchange.com/docs/wrapper
_`[4]`: https://github.com/openstack/cinder/blob/master/cinder/db/sqlalchemy/api.py#L2324
_`[5]`: https://github.com/openstack/api-wg/blob/64e3e9b07272f50353429dc51d98524642ab6d67/guidelines/counting.rst





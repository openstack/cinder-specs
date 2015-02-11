..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Generic filter support for volume queries
==========================================

https://blueprints.launchpad.net/cinder/+spec/db-volume-filtering

The filtering support of the volume DB APIs is inconsistent. For example,
filtering is supported when querying all volumes and when querying volumes
by project. However, filtering is not supported when querying volumes by host
or by group.


Problem description
===================

DB functions exist to get all volumes, to get all volumes in a particular
project, to get all volumes in a particular group, and to get all volumes
hosted on a particular host. See the following functions in the DB API:

* volume_get_all
* volume_get_all_by_project
* volume_get_all_by_group
* volume_get_all_by_host

Only the queries that get all volumes and that get all volumes by project
support additional filtering.

The purpose of this blueprint is to make the filtering support consistent
across these APIs.


Proposed change
===============

The current Volume filtering logic is already encapsulated in the common
_generate_paginate_query function. The filtering needs to be moved into
a common function (something like _process_volume_filters) that would
update a model query object with the filter information.

Then, for example, the volume_get_all_by_host could utilize it as follows::

    def volume_get_all_by_host(context, host, filters=None):
        """Retrieves all volumes hosted on a host."""
        if host and isinstance(host, basestring):
            session = get_session()
            with session.begin():
                host_attr = getattr(models.Volume, 'host')
                conditions = [host_attr == host,
                              host_attr.op('LIKE')(host + '#%')]
                query = _volume_get_query(context).filter(or_(*conditions))
                if filters:
                    query = _process_volume_filters(query, filters)
                    if not query:
                        return None
                return query.all()
        elif not host:
            return []

Alternatives
------------

Instead of adding filter support to the other volume APIs, a caller could
simply invoke the volume_get_all API with a filter that defines the host or
the group information. The downside of this approach is that is requires the
caller to understand how to form that query; this is especially problematic
for the host API since the filter is actually an OR of:

* A exact string match for the given host
* A REGEX match where the host matches a value in the form of "<host>#<pool>"

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

Complicated DB filters could affect query performance; however, this already
exists for the volume_get_all and the volume_get_all_by_project APIs.

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
  Steven Kaufer (kaufer)

Other contributors:
  None

Work Items
----------

* Refactor Volume filter logic in the sqlalchemy DB API into a common function
  and invoke it from the existing _generate_paginate_query function
* Update the functions to get volumes by host and by group to use the common
  filtering function


Dependencies
============

None


Testing
=======

Since common filter processing will be used for all volume DB queries, the
existing test coverage is sufficient.


Documentation Impact
====================

None


References
==========

None

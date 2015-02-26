..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cinder Pagination Sorting Enhancements
==========================================

https://blueprints.launchpad.net/cinder/+spec/cinder-pagination

Currently, the sorting support for Cinder allows the caller to specify a
single sort key and sort direction. This blueprint enhances the sorting
support for the /volumes and /volumes/detail APIs so that multiple sort keys
and sort directions can be supplied on the request.


Problem description
===================

There is no support for retrieving volume data based on multiple sort keys; a
single sort key and direction is currently supported and it is defaulted to
descending sort order by the "created_at" key. In order to retrieve data in
any sort order and direction, the REST APIs need to accept multiple sort keys
and directions.

Use Case: A UI that displays a table with only the page of data that it
has retrieved from the server. The items in this table need to be sorted
by status first and by display name second. In order to retrieve data in
this order, the APIs must accept multiple sort keys/directions.


Proposed change
===============

The /volumes and /volumes/detail APIs will align with the API working group
guidelines (see References section) for sorting and support the following
parameter on the request:

* sort: Comma-separated list of sort keys, each key is optionally appended
  with <:dir>, where 'dir' is the direction for the corresponding sort key
  (supported values are 'asc' for ascending and 'desc' for descending)

For example:
``/volumes?sort=status:asc,display_name:asc,created_at:desc``

Note: The "created_at" and "id" sort keys are always appended at the end of
the key list if they are not already specified on the request.

The database layer already supports multiple sort keys and directions. This
blueprint will update the API layer to retrieve the sort information from
the API request and pass that information down to the database layer.

All sorting is handled in the cinder.common.sqlalchemyutils.paginate_query
function.  This function accepts an ORM model class as an argument and the
only valid sort keys are attributes on the given model class.  Therefore,
the valid sort keys are limited to the model attributes on the
models.Volume class.

Alternatives
------------

Multiple sort keys and directions could be passed using repeated 'sort_key'
and 'sort_dir' query parameters. For example:

``/volumes?sort_key=status&sort_dir=asc&sort_key=display_name&sort_dir=asc&
sort_key=created_at&sort_dir=desc``

However, this is not aligned with the API sorting guidelines.

Also, this additional sorting could be conditionally enabled based on the
existence of a new extension; however, since sorting by a single key is
already supported, then a new extension may not be needed to enhance this
support to multiple keys/directions. However, an extension could be created
if necessary.

Data model impact
-----------------

None

REST API impact
---------------

The following existing v2 GET APIs will support the new sorting parameters:

* /v2/{tenant_id}/volumes
* /v2/{tenant_id}/volumes/detail

Note that the design described in this blueprint could be applied to other GET
REST APIs but this blueprint is scoped to only those listed above. Once this
design is finalized, then the same approach could be applied to other APIs.

The existing API documentation needs to be updated to include the following
new Request Parameters:

+-----------+-------+--------+------------------------------------------------+
| Parameter | Style | Type   | Description                                    |
+===========+=======+========+================================================+
| sort      | query | string | Comma-separated list of sort keys and optional |
|           |       |        | sort directions in the form of key<:dir>,      |
|           |       |        | where 'dir' is either 'asc' for ascending      |
|           |       |        | order or 'desc' for descending order. Defaults |
|           |       |        | to the 'created_at' and 'id' keys in           |
|           |       |        | descending order.                              |
+-----------+-------+--------+------------------------------------------------+

Currently, the volumes query supports the 'sort_key' and 'sort_dir' parameters;
these will be deprecated. The API will raise a "badRequest" error response
(code 400) if both the new 'sort' parameter and a deprecated 'sort_key' or
'sort_dir' parameter is specified.

Neither the API response format nor the return codes will be modified, only
the order of the volumes that are returned.

In the event that an invalid sort key is specified then a "badRequest" error
response (code 400) will be returned with a message like "Invalid input
received: Invalid sort key".

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

The cinderclient should be updated to accept sort keys and sort directions,
using the 'sort' parameter being proposed in the cross-project spec:
https://review.openstack.org/#/c/145544/

Performance Impact
------------------

All sorting will be done in the database. The choice of sort keys is limited
to attributes on the models.Volume ORM class -- not every attribute key
returned from a detailed query is a valid sort key.

Performance data was gathered by running on a simple devstack VM with 2GB
memory. 5000 volumes were inserted into the DB. The data shows that the
sort time on the main data table is dwarfed (see first table below) when
running a detailed query -- most of the time is spent querying the the other
tables for each item; therefore, the impact of the sort key on a detailed
query is minimal.

For example, the data below compares the processing time of a GET request for
a non-detailed query to a detailed query with various limits using the default
sort keys. The purpose of this table is to show the the processing time for a
detailed query is dominated by getting the additional details for each item.

+-------+--------------------+----------------+---------------------------+
| Limit | Non-Detailed (sec) | Detailed (sec) | Non-Detailed / Detailed % |
+=======+====================+================+===========================+
| 50    | 0.0560             | 0.8621         | 6.5%                      |
+-------+--------------------+----------------+---------------------------+
| 100   | 0.0813             | 1.6839         | 4.8%                      |
+-------+--------------------+----------------+---------------------------+
| 150   | 0.0848             | 2.4705         | 3.4%                      |
+-------+--------------------+----------------+---------------------------+
| 200   | 0.0874             | 3.2502         | 2.7%                      |
+-------+--------------------+----------------+---------------------------+
| 250   | 0.0985             | 4.1237         | 2.4%                      |
+-------+--------------------+----------------+---------------------------+
| 300   | 0.1229             | 4.8731         | 2.5%                      |
+-------+--------------------+----------------+---------------------------+
| 350   | 0.1262             | 5.6366         | 2.2%                      |
+-------+--------------------+----------------+---------------------------+
| 400   | 0.1282             | 6.5573         | 2.0%                      |
+-------+--------------------+----------------+---------------------------+
| 450   | 0.1458             | 7.2921         | 2.0%                      |
+-------+--------------------+----------------+---------------------------+
| 500   | 0.1770             | 8.1126         | 2.2%                      |
+-------+--------------------+----------------+---------------------------+
| 1000  | 0.2589             | 16.0844        | 1.6%                      |
+-------+--------------------+----------------+---------------------------+

Non-detailed query data was also gathered. The table below compares the
processing time using default sort keys to the processing using display_name
as the sort key. Items were added with a 40 character display_name that was
generated in an out-of-alphabetical sort order.

+-------+--------------------+------------------------+------------+
| Limit | Default keys (sec) | display_name key (sec) | Slowdown % |
+=======+====================+========================+============+
| 50    | 0.0560             | 0.0600                 | 7.1%       |
+-------+--------------------+------------------------+------------+
| 100   | 0.0813             | 0.0832                 | 2.3%       |
+-------+--------------------+------------------------+------------+
| 150   | 0.0848             | 0.0879                 | 3.7%       |
+-------+--------------------+------------------------+------------+
| 200   | 0.0874             | 0.0906                 | 3.7%       |
+-------+--------------------+------------------------+------------+
| 250   | 0.0985             | 0.1031                 | 4.7%       |
+-------+--------------------+------------------------+------------+
| 300   | 0.1229             | 0.1198                 | -2.5%      |
+-------+--------------------+------------------------+------------+
| 350   | 0.1262             | 0.1319                 | 4.5%       |
+-------+--------------------+------------------------+------------+
| 400   | 0.1282             | 0.1368                 | 6.7%       |
+-------+--------------------+------------------------+------------+
| 450   | 0.1458             | 0.1458                 | 0.0%       |
+-------+--------------------+------------------------+------------+
| 500   | 0.1770             | 0.1619                 | -8.5%      |
+-------+--------------------+------------------------+------------+
| 1000  | 0.2589             | 0.2659                 | 2.7%       |
+-------+--------------------+------------------------+------------+

In conclusion, the sort processing on the main data table has minimal impact
on the overall processing time. For a detailed query, the sort time is dwarfed
by other processing -- even if the sort time when up 3x it would only
represent 4.8% of the total processing time for a detailed query with a limit
of 1000 (and only increase the processing time by .11 sec with a limit of 50).

Other deployer impact
---------------------

The choice of sort keys has a minimal impact on data retrieval performance
(see performance data above). Therefore, the user should be allowed to
retrieve data in whatever order they need to for creating their views (see
use case in the Problem Description).

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kaufer (kaufer@us.ibm.com)

Other contributors:
  None

Work Items
----------

Ideally the logic for processing the sort parameters would be common to all
components and would be done in oslo; a similar blueprint is also being
proposed in nova:
https://blueprints.launchpad.net/nova/+spec/nova-pagination

Therefore, I see the following work items:

* Duplicate the common code being proposed in nova to process the sort
  parameters, see https://review.openstack.org/#/c/95260/. Once both projects
  are using the same code then it should be moved into oslo.
* Update the API to retrieve the sort information and pass down to the
  DB layer (requires changes to volume/api.py, db/api.py, and
  db/sqlalchemy/api.py)
* Update the cinderclient to accept and process multiple sort keys and sort
  directions


Dependencies
============

* Related (but independent) change being proposed in nova:
  https://blueprints.launchpad.net/nova/+spec/nova-pagination
* CLI Sorting Argument Guidelines cross project spec:
  https://review.openstack.org/#/c/145544/

Testing
=======

Both unit and Tempest tests need to be created to ensure that the data is
retrieved in the specified sort order. Tests should also verify that the
default sort keys ("created_at" and "id") are always appended to the user
supplied keys (if the user did not already specify them).

Testing should be done against multiple backend database types.


Documentation Impact
====================

The /volumes and /volumes/detail API documentation will need to be updated
to:

- Reflect the new sorting parameters and explain that these parameters will
  affect the order in which the data is returned.
- Explain how the default sort keys will always be added at the end of the
  sort key list

The documentation could also note that query performance will be affected by
the choice of the sort key, noting which keys are indexed.


References
==========

API Working group sorting guidelines:
https://github.com/openstack/api-wg/blob/master/guidelines/
pagination_filter_sort.rst

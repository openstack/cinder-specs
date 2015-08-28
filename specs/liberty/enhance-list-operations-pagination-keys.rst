..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Enhance List Operations with Pagination Keys
============================================

https://blueprints.launchpad.net/cinder/+spec/extend-limit-implementations

Currently, the pagination keys, like limit, marker, sort key and sort
direction and the next link generation, have been implemented for the volume
list operations, but not available in other list operations. This blueprint
implements the pagination keys and the next link generation for other items,
like snapshots, volume transfers, consistency group, consistency group
snapshots and backups.


Problem description
===================

The pagination keys mentioned in the above section have been implemented in
the list operations for volumes. This feature can be extended to other cinder
concepts. The reason why snapshots, volume transfers, consistency groups,
consistency group snapshots, and backups are being targeted is because these
entities have relatively high frequencies to query and it is more probable for
them to reach a large amount comparing to other entities, like type,
availability zone, etc.

The next link generation is already available in volume list, so it is
straightforward to implement it for other cinder concepts. Please refer to
the _get_collection_links function in the cinder.api.common.ViewBuilder class.


Use Cases
=========

Suppose the maximum page size 1000 and there are more than 1000 volume
snapshots. If there is no pagination key marker implemented for
snapshot, the maximum snapshots the user can query from the API is 1000. The
only way to query more than 1000 snapshots is to increase the default maximum
page size, which is a limitation to the cinder list operations. If no next link
is generated, it is not possible to retrieve more items than the maximum page
size. Even if the next link is generated in this case, it will still return
the same results as the previous request without the key marker honored. In
conclusion, both the pagination keys and the next link generation are necessary
to add for the cinder entities.


Proposed change
===============

The change depends on some work items, which have been put in the
"Dependencies" section.

The list APIs for snapshots, volume transfers, consistency group, consistency
group snapshots and backups will support the following parameters:

* limit: Key used to determine the number of returned items in the response
* marker: Key used to determine the item, where to start the query
* sort: This key is a comma-separated list of sort keys. Sort directions
  can optionally be appended to each sort key, separated by the ':' character

Multiple sort keys and sort directions will be implemented as they are done
for volumes in the patch "GET volumes API sorting REST/volume/DB updates",
https://review.openstack.org/#/c/141915/.

The change is compliant with the syntax for client sorting arguments defined
in the patch "CLI Sorting Argument Guidelines",
https://review.openstack.org/#/c/145544/.

The caller can specify these parameters in the request to query the list of
the items,
e.g. /snapshots?limit=5&marker=snapshot_id&sort=key1:asc,key2:desc,key3:asc

If there are more items available to query in further requests, the next link
should be correctly generated for snapshots, volume transfers, consistency
group, consistency group snapshots and backups.


Alternatives
------------

It is possible that we define different pagination keys, other than limit,
marker, sort information to implement the pagination for other items
in Cinder.

However, to keep the consistency in code change, it is better to follow the
pattern that volume list does.

Another alternative is cinder sticks with no pagination for other entities
except volumes. However, the default maximum page size 1000 can be exceeded,
by some entities, like snapshots, in the production environment. It will
cause issues in other entities in future.


Data model impact
-----------------

None

REST API impact
---------------

The following existing v2 GET APIs will support the new sorting parameters:

* /v2/{tenant_id}/{items}
* /v2/{tenant_id}/{items}/detail

{items} will be replaced by the appropriate entities as follows:

* For snapshots:
  * /v2/{tenant_id}/snapshots
  * /v2/{tenant_id}/snapshots/detail

* For volume transfers:
  * /v2/{tenant_id}/os-volume-transfer
  * /v2/{tenant_id}/os-volume-transfer/detail

* For consistency group:
  * /v2/{tenant_id}/consistencygroups
  * /v2/{tenant_id}/consistencygroups/detail

* For consistency group snapshots:
  * /v2/{tenant_id}/cgsnapshots
  * /v2/{tenant_id}/cgsnapshots/detail

* For backups:
  * /v2/{tenant_id}/backups
  * /v2/{tenant_id}/backups/detail

The existing API needs to support the following new Request Parameters for
the above cinder concepts:

+-----------+-------+---------+---------------------------------------------+
| Parameter | Style | Type    | Description                                 |
+===========+=======+=========+=============================================+
| limit     | query | integer | Key used to determine the number of         |
|           |       |         | returned items in the response              |
+-----------+-------+---------+---------------------------------------------+
| marker    | query | string  | Key used to determine the item, where       |
|           |       |         | to start the query                          |
+-----------+-------+---------+---------------------------------------------+
| sort      | query | list    | A comma-separated list of sort keys, with   |
|           |       |         | the sort directions optionally appended to  |
|           |       |         | each sort key                               |
+-----------+-------+---------+---------------------------------------------+

In the event that an invalid pagination key is specified then a "badRequest"
error response (code 400) will be returned.

* For the limit key, it returns a message like "Invalid input received:
  Invalid limit key".

* For the marker key, it returns a message like "Invalid input received:
  Invalid marker key".

* For the sort information, it returns a message like "Invalid input received:
  Invalid sort key" or "Invalid input received: Invalid sort direction"
  depending on different circumstances.

The next link will be put in the response returned from cinder if it is
necessary.

* For snapshots, it replies:
{
    "snapshots": [<List of snapshots>],
    "snapshots_links": [{'href': '<next_link>', 'rel': 'next'}]
}

* For volume transfers, it replies:
{
    "transfers": [<List of transfers>],
    "transfers_links": [{'href': '<next_link>', 'rel': 'next'}]
}

* For consistency group, it replies:
{
    "consistencygroups": [<List of consistencygroups>],
    "consistencygroups_links": [{'href': '<next_link>', 'rel': 'next'}]
}

* For consistency group snapshots, it replies:
{
    "cgsnapshots": [<List of cgsnapshots>],
    "cgsnapshots_links": [{'href': '<next_link>', 'rel': 'next'}]
}

* For backups, it replies::
{
    "backups": [<List of backups>],
    "backups_links": [{'href': '<next_link>', 'rel': 'next'}]
}


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

The cinderclient should be updated to accept limit, marker, sort information,
in snapshots, volume transfers, consistency group, consistency group snapshots
and backups.

The user will be able to specify pagination keys, like limit, marker, sort
information to list snapshots, volume transfers, consistency group, consistency
group snapshots and backups.


Performance Impact
------------------

None

Other deployer impact
---------------------

The deployer should be aware that the flag osapi_max_limit can set the maximum
number of items that a collection resource returns in ONE single response, but
it will not limit the number of items returned for the cinderclient request any
longer.

After the pagination keys and the next link generation are implemented for the
cinder entities, the cinderclient request can retrieve more items than the flag
osapi_max_limit sets, because it can fetch the items multiple times via the
next link with the marker key until all the items are returned if no limit key
is set or the number of items equals to limit if the limit key is set.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Vincent Hou (sbhou@cn.ibm.com)

Other contributors:
  None

Work Items
----------

Since the logic code of next link generation is finished in a common class. We
do not need to repeat the work any more.

Therefore, I see the following work items:

* Add the code to call the common functions to get the limit, marker and
  sort parameters processed for snapshots, volume transfers,
  consistency group, consistency group snapshots and backups.
* Add the code to do the database query with the paginations keys for
  snapshots, volume transfers, consistency group, consistency group snapshots
  and backups.
* Add the support of the paginations keys for snapshots, volume transfers,
  consistency group, consistency group snapshots and backups in cinderclient.
* Modify the existing APIs to support passing the limit, marker, and sort
  information from the API layer to the database layer.
* Unit tests to ensure that these pagination keys and the next link generation
  is supported for snapshots, volume transfers, consistency group, consistency
  group snapshots and backups.

Dependencies
============

* Cinder pagination:
  https://blueprints.launchpad.net/cinder/+spec/cinder-pagination, accepted
* GET volumes API sorting REST/volume/DB updates:
  https://review.openstack.org/#/c/141915/, WIP
* CLI Sorting Argument Guidelines: https://review.openstack.org/#/c/145544/,
  accepted
* Server sorting guidelines: https://review.openstack.org/#/c/145579, merged


Testing
=======

Both unit and Tempest tests need to be created to ensure that snapshots, volume
transfers, consistency group, consistency group snapshots and backups support
the pagination keys of limit, marker, and sort information, and the next link
generation is available if necessary.


Documentation Impact
====================

The API documentation will need to be updated to accept limit, marker,
sort key and sort direction, in snapshots, volume transfers, consistency
group, consistency group snapshots and backups as it does for volumes.


References
==========

None

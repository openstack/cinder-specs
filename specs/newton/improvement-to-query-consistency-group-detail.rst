..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Improvement to query consistency group detail
=============================================

https://blueprints.launchpad.net/cinder/+spec/improvement-to-query-consistency-group-detail

Return the volume id list if they exist in CG when querying CG detail.

Problem description
===================

Currently, querying consistency group detail will not return list of volumes
ids that exist in this CG, end users don't know how many and which volumes are
in it. They must query the volume detail one by one to get the CG's id and
form the list of volume ids. It's very inconvenient to user experience.


Use Cases
=========

After adding volumes to a consistency group, end user wants to know how many
and which volumes are in a CG before they decide to make CG snapshot.
According to this feature, they can get this information easily by querying
consistency group detail, and make further adjustment, like adding more
volumes, or remove some volumes from this consistency group.

Proposed change
===============

1.Add volume id list into response body of querying a consistency group detail
if user wants those volume's information.

2.Add group_id query filter to /volumes to get a list of volumes
in a CG. For example "cinder list --group-id <uuid of CG>".

NOTE: Since we're working on Generic Volume Group[1], to avoid unnecessary
code mirgation later, we will implement the 'group_id' query filter first,
and after Generic Volume Group is merged, we will implement the #1 way
dependent on the Generic Group spec.

Alternatives
------------

None

Data model impact
-----------------

For performance impact, if we want to add index to consistencygroup_id column,
there is data model impact.

REST API impact
---------------

* Add volume id list into response body of querying a CG detail if specifying
the argument 'list_volume=True', this will be dependent on the Generic Group
spec now, so just leave a example here::

GET /v3/{project_id}/consistencygroups/{consistency_group_id}?list_volume=True
RESP BODY: {"consistencygroup": {"status": "XXX",
                                 "description": "XXX",
                                 ...,
                                 "volume_list":['volume_id1',
                                                ...,
                                                'volume_idn']
                                }
            }

* Add a filter "group_id=xxx" in URL of querying volume list/detail::

GET /v3/{project_id}/volumes?group_id=XXX

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

None

Performance Impact
------------------

There is no performance impact if user is NOT using 'list_volume=True'.
We just need to consider to add a additional DB query to get all
volume ids which have same CG id. So the performance impact should
be small with 'list_volume=True'. If considering large scale volumes,
we can add index to consistencygroup_id column of volume table
to reduce performance impact.

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

* Implement code in db query and add list to response body.
* Update cinderclient to support this function.
* Add change to API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change".


Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.

References
==========

[1]https://review.openstack.org/#/c/303893/

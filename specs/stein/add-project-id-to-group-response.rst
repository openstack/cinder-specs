..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add project_id attribute to group response
==========================================
https://blueprints.launchpad.net/cinder/+spec/add-project-id-to-group-response

This blueprint proposes to add ``project_id`` attribute to
the response body of list groups with detail and show group detail APIs.

Problem Description
===================

Currently, the group show response doesn't include project_id.
It is an important parameter while differentiating between multiple groups
in horizon.

Use Cases
=========

As horizon is adding a tab for group listing with detail in the admin panel[1],
it requires project_id as a response parameter from the groups API.
It is similar to what is implemented in volumes and snapshots list.

Proposed change
===============

This spec proposes to add ``project_id`` attribute to the
response body of list groups with detail and show group detail APIs.

Add a new microverion API to add ``project_id`` attribute
to the response body of list groups with detail and show group detail APIs:

- List groups with detail GET /v3/{project_id}/groups/detail

- Show group detail GET /v3/{project_id}/groups/{group_id}

Alternatives
------------

None

REST API impact
---------------

Add a new microversion in Cinder API.

* List groups with detail::

    GET /v3/{project_id}/groups/detail
    Response BODY:
    {
        "groups": [{
            ...
            "project_id": "7ccf4863071f44aeb8f141f65780c51b"
        }]
    }

* Show group detail::

    GET /v3/{project_id}/groups/{group_id}
    Response BODY:
    {
        "groups": [{
            ...
            "project_id": "7ccf4863071f44aeb8f141f65780c51b"
        }]
    }

Calling this method shows ``project_id`` for a group.
It is intended for admins to use, which is used to display the project_id to which
the group belongs, and controlled by ``GROUP_ATTRIBUTES_POLICY``.

Data model impact
-----------------

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
  Rajat Dhasmana <rajatdhasmana@gmail.com>

Work Items
----------

* Add a new microversion.
* Add ``project_id`` to the response body of list groups
  with detail and show group detail APIs.
* Add the related unit tests.
* Update related list groups with detail and show group detail api doc.

Dependencies
============

None

Testing
=======

* Unit-tests, tempest and other related test should be implemented

Documentation Impact
====================

None

References
==========

[1] https://review.openstack.org/#/c/624599/

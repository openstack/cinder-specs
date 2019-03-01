..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Add project_id attribute to group and group snapshot response
=============================================================
https://blueprints.launchpad.net/cinder/+spec/add-project-id-to-group-groupsnapshot-response

This blueprint proposes to add ``project_id`` attribute to the response
body of list groups with detail, list group snapshots with detail,
show group detail and show group snapshot detail APIs.

Problem Description
===================

Currently, the group and group snapshot show response doesn't include
project_id.
It is an important parameter while differentiating between multiple groups
and multiple group snapshots in horizon.

Use Cases
=========

As horizon is adding tabs for groups and group snapshots listing
with detail in the admin panel[1], it requires project_id as a
response parameter from the groups and group snapshots API.
It is similar to what is implemented in volumes and snapshots list.

Proposed change
===============

This spec proposes to add ``project_id`` attribute to the response
body of list groups with detail, list group snapshots with detail,
show group detail and show group snapshot detail APIs.

Add a new microverion API to add ``project_id`` attribute to the
response body of list groups with detail, list group snapshots with
detail, show group detail and show group snapshot detail APIs:

- List groups with detail GET /v3/{project_id}/groups/detail

- List group snapshots with detail GET /v3/{project_id}/group_snapshots/detail

- Show group detail GET /v3/{project_id}/groups/{group_id}

- Show group snapshot detail GET /v3/{project_id}/group_snapshots/{group_snapshot_id}

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

* List group snapshots with detail::

    GET /v3/{project_id}/group_snapshots/detail
    Response BODY:
    {
        "group_snapshots": [{
            ...
            "project_id": "7ccf4863071f44aeb8f141f65780c51b"
        }]
    }

* Show group detail::

    GET /v3/{project_id}/groups/{group_id}
    Response BODY:
    {
        "group": [{
            ...
            "project_id": "7ccf4863071f44aeb8f141f65780c51b"
        }]
    }

* Show group snapshot detail::

    GET /v3/{project_id}/group_snapshots/{group_snapshot_id}
    Response BODY:
    {
        "group_snapshot": [{
            ...
            "project_id": "7ccf4863071f44aeb8f141f65780c51b"
        }]
    }

Calling this method shows ``project_id`` for a group or group snapshot.
It is intended for admins to use, which is used to display the project_id to which
the group or group snapshot belongs, and controlled by ``GROUP_ATTRIBUTES_POLICY``
or ``GROUP_SNAPSHOT_ATTRIBUTES_POLICY``.

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
* Add ``project_id`` to the response body of list groups with detail,
  list group snapshots with detail, show group detail and show group
  snapshot detail APIs.
* Add the related unit tests.
* Update related list groups with detail, list group snapshots with
  detail, show group detail and show group snapshot detail api doc.

Dependencies
============

None

Testing
=======

* Unit-tests and other related test should be implemented

Documentation Impact
====================

None

References
==========

[1] https://blueprints.launchpad.net/horizon/+spec/cinder-generic-volume-groups

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================================
Reset generic volume group and group snapshot statuses
===========================================================================

https://blueprints.launchpad.net/cinder/+spec/reset-cg-and-cgs-status


Problem description
===================

Currently we support the administrator to reset the status of volume,
snapshot, backup and so on, but both of the generic volume group and group
snapshot could also run into error status, the administrator could only
modify the status in database. This is inconvenient for administrator.

Use Cases
=========

The primary use case is for the manual recovery of some sort of issue with
generic volume group's status(snapshot's inclusive). This is helpful for
recovering the two resources from a bad status due to real issues with the
system or crashed from bugs.

Proposed change
===============

Take the generic volume group for example, The change can add a new admin
generic group action on the resource. The logic could be the same as
volume's action ``reset_status`` which already exists, for example,
to reset the status of generic group to error, it would be::

    POST /v3/{tenant_id}/groups/{group_id}/action
    {
        'reset_status': {
           'status': 'error'
        }
    }

The 'status' should be one of the generic group status,the api will validate
the input status and then update the status in database.

Alternatives
------------

We could just leave it as a manual db operation, but that leaves it up to an
admin to go manually update the db to achieve the same goal.

Data model impact
-----------------

None

REST API impact
---------------

The API microversion will have to be bumped for the new APIs below.

New GroupAdminController::

    POST /v3/{tenant_id}/groups/{group_id}/action

And new admin action for it::

    reset_status

New GroupSnapshotAdminController::

    POST /v3/{tenant_id}/group_snapshots/{group_snapshot_id}/action

And new admin action for it::

    reset_status

They both required the identical request body, and 'status' should be
passed in as a string::

    {
        'reset_status': {
           'status': 'error'
        }
    }

For the return codes if the status is not valid we will return a 400 Bad
Request, if the resource is not found it will be a 404 Not Found, and will
otherwise return a 202.

Cinder-client impact
--------------------

The cinder-client will add commands to expose these new APIs(use 'state'
instead of 'status' to keep consistency with existed reset commands):

New command to reset generic group status:

    group-reset-state <group> --state <state>

the 'group' should be name or ID of the specified group.

New command to reset group snapshot status:

    group-snapshot-reset-state <group-snapshot> --state <state>

the 'group-snapshot' should be name or ID of the specified group snapshot.

Security impact
---------------

None

Notifications impact
--------------------

We will add ``reset_group_status`` and ``reset_group_snapshot_status``
notifications that will behave the same way as the ``reset_status``
notifications. There will be a ``start`` and ``end`` notification.

Other end user impact
---------------------

The new API will be exposed to users via the python-cinderclient.

Performance Impact
------------------

While it is doing a db operation it should be a very low impact.

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
  tommylikehu(TommyLikeHu@gmail.com)

Work Items
----------

* Implement new admin action and tests in Cinder.
* Add support in python-cinderclient.


Dependencies
============

None


Testing
=======

* Unit tests in Cinder.
* Tempest tests in Cinder.
* Functional tests in Cinder.
* Unit tests in python-cinderclient.


Documentation Impact
====================

New admin docs to explain how to use the API.


References
==========

None
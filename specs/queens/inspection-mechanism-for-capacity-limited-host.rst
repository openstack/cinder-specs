..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Inspection Mechanism For Capacity Limited Host
==============================================

https://blueprints.launchpad.net/cinder/+spec/inspection-mechanism-for-capacity-limited-host

Cinder now has a scheduler filter called ``CapacityFilter`` which allow users
to create volume under capacity limit. But some APIs which don't go through
cinder-scheduler will break the destination host's capacity.


Problem description
===================

Some backends now support the thin provisioning capability. But only Cinder
gets and checks this capability in cinder-scheduler's filter: CapacityFilter.
It means that some APIs which go to the backend directly without
cinder-scheduler doesn't get and check the ``reserved_percentage`` and the
``max_over_subscription_ratio`` of the backend. Then it may break the backend's
capacity. Such as "extend volume", "create volume from snapshot", "copy volume"
and so on.


Use Cases
=========

Usually, the capacity is a hard limit for the specified backend when Cinder
works with the CapacityFilter (It is one of the default filters in Cinder). It
means that every action to this kind of backend should keep its capacity. We
should not break the capability of the backend which support the thin
provisioning.


Proposed change
===============

- Add some common functions to check the backend's capacity at the
  cinder-volume layer. The patch[1] is a POC to solve volume creation problem.
- The APIs which 1) change the backend's capacity and, 2) don't go through
  cinder-scheduler, should be directly checked. It includes:

  * POST /v3/{project_id}/volumes/{volume_id}/action  (os-extend)  (SOLVED)[2]
  * POST /v3/{project_id}/volumes  (create from snapshot, volume or replica)
  * POST /v3/{project_id}/groups/action (create_from_src)
  * POST /v3/{project_id}/snapshots

NOTE:

- Create volume requests for a snapshot, volume, or replica will change the
  backend's capacity, and will therefore cause the volume's status to switch to
  "error".
- The cinder admin isn't required to use the ``CapacityFilter``, since the
  volume capacity limit doesn't need to be checked at the driver layer.

Alternatives
------------

For the APIs which will change the backend's capacity and not go through
cinder-scheduler, we could route them through the scheduler instead of jumping
right to a volume-service.

The problem for the extend API is solved by a bug fix[2] recently.
In the patch, it passes a "volume" parameter which includes the host name to
cinder-scheduler to indicate that the destination host is specified. Then
cinder-scheduler will only check the specified host with
``backend_passes_filters`` function. If no error raises, it means that the POST
action is allowed.

NOTE:

- We need to bump the RPC call's version between cinder-api and
  cinder-scheduler to keep backward compatibility for each related API.
- The create actions will always call through the cinder-scheduler, even if
  the destination host is specified. If the cinder-scheduler can't match the
  new RPC version, the old behavior is kept.

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

Instead of sending notifications, we'd better send user messages if the actions
fail.

Other end user impact
---------------------

The create requests which will exceed the backend's capacity will make the
volume's status to "error".

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
  wangxiyuan <wangxiyuan@huawei.com>
  Huang Zhiteng <winston.d@gmail.com>
  Erlon R. Cruz <sombrafam@gmail.com>
  tommylikehu<tommylikehu@gmail.com>

Work Items
----------

- Add some common check functions and call them in:

  * create volume from volume, snapshot or replica
  * create cg from source
  * create group from source
  * create snapshot

- Add and update the unit test.


Dependencies
============

None


Testing
=======

Unit tests will be added and updated.


Documentation Impact
====================

None


References
==========

[1] Huang Zhiteng: https://review.openstack.org/#/c/437677/
[2] Erlon R.Cruz: https://review.openstack.org/#/c/405578/

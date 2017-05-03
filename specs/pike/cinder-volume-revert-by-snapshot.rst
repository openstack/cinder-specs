..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Cinder volume revert to snapshot
================================

https://blueprints.launchpad.net/cinder/+spec/revert-volume-to-snapshot

It makes sense to be able to revert a volume to a previously taken
snapshot. The revert function can be used to revert a volume to a previous
snapshot, restoring the volume to the state at the time the snapshot was
created. The feature supports reverting a volume to the most recent snapshot
taken, also manila (shared file system service) has already supported what
we are proposing in Ocata `[1]`_.

The revert process will overwrite the current state and data of the volume.
If the volume was extended after the snapshot, the request would be rejected
 (reason is described in proposed change section). It's assumed that the user
doing a revert is discarding the actual state and data of the volume.

The purpose of this feature is to give users the possibility of recover
instances and volumes more conveniently and with minimal downtime.

Problem description
===================

Currently, a user can recover an instance by creating a volume from specified
snapshot and attaching it using the following workflow:

1. User creates a volume(s)
2. User attach volume to instance
3. User creates a snapshot
4. Something happens (crash) causing the need of a revert
5. User creates a volume(s) from the snapshot(s)
6. User detach old volumes
7. User attach new volumes

This workflow has the following problems:

* To create a volume from a snapshot, some of vendors need to copy data to the
  new volume. This process takes more time than simply reverting to the most
  recent snapshot.

* To create a volume from a snapshot, the network configuration will change
  and that leads to problems in most of applications demanding admin
  intervention.

* If the instance was created from a boot volume, initial customization
  scripts might not be able to configure the instance to the same status it
  was before the crash.

* It consumes quota from the user and in some cases real space on the backend.

Another alternative is to restore the data from a backup, but that process
can't be done by the cloud user, it's not a common case (data inconsistency
is more likely the main cause), also, the end user would expect a less time
consuming operation.

As mentioned above, we hope that can offer a way to recover instance quickly
using an easiest way, revert snapshot may be a good choice.

Use Cases
=========

The primary use case is the situation when an instance or volume crashes due
to any adverse situation, attack or malfunctioning while storage data is kept
alive. In this situation, the following workflow will be used:

1. User creates a volume.
2. User attaches the volume to instance.
3. User creates snapshots of safe points.
4. Something happens causing the need of a revert.
5. User detaches the volume.
6. User reverts to the safe snapshot(s).

Proposed change
===============

Theoretically, if multiple snapshots exist, a storage system could restore
a volume to any one of those snapshots. But revert a volume to a
snapshot which is not the latest one typically deletes the later snapshots,
which is a form of data loss and may not be what the user intends. So this
feature will limit cinder to revert the most recent volume snapshot known
to cinder.

For this use case, reverting backwards in time starting with the most recent
snapshot makes sense. Only the user can determine whether a volume is usable
or must be reverted, so getting back to a good state is a manual iterative
process::

  Observe corruption in volume
  Revert to most recent snapshot
  While corruption is present in volume:
      Delete most recent snapshot
      Revert to most recent snapshot

A suggestion with simple API which merely include the volume ID and allow
cinder to determine the most recent snapshot has been put forward, but it
could led to potential concurrency bugs:

1. If two users call volume-revert-to-snapshot and snapshot-create at the
   same time, the two calls race and one user could be surprised about
   which snapshot was used to revert the volume.
2. if two users call volume-revert-to-snapshot and snapshot-delete at the
   same time, the calls race and someone may be surprised when the volume
   is reverted to an even earlier snapshot than expected (which is a form
   of data loss).

To keep the REST interface explicit, and to avoid the race conditions
described above, the interface will accept the ID of the snapshot to be
restored. cinder will verify that the snapshot is the most recent one via
the created_at field on the snapshot object, returning an error if not. At
no time will cinder delete any snapshots during the revert operation, and
it will not operate on any snapshot other than the one specified in the REST
call.

There are two cases when reverting a volume, one is detached volume, and
the other is the attached volume. We would only support reverting a
detached volume.

As we support extend volumes at present, we could meet the case that
snapshot is smaller than volume when reverting to snapshot and it's obviously
safe to revert in that case. But we will still restrict to revert the volume
which current volume size is equal to snapshot, cause we don't support shrink
volume yet `[2]`_ (that ability will be used in the generic method, if the
driver don't support revert to snapshot).

Therefore, we need to do the following changes in order to support volume
revert to snapshot.

- **Add volume revert to snapshot API**.

  We will add a revert to snapshot API, in the API layer, the validations (
  existence, status and size) on both snapshot and volume will be proceeded
  along with whether the snapshot is the latest one.

  Cinder will accept the request only if the volume's status is 'available'.

- **Implement revert to snapshot in cinder volume service**.

  During a snapshot restoration, there are two objects that are
  affected whose status must be reflected. The volume's status is
  'reverting', the snapshot's status is also 'restoring'. Other
  operations must not be allowed to either object during the
  restoration.

  If the cinder driver can not support the revert feature, it will
  use the generic implementation to run revert process.

  After a successful restoration, both objects' status will again be set
  to the status before reverting.  If the restoration fails, the volume
  will be set to 'error', and the snapshot will be set to 'available'.

- **Create snapshot for backup**

  The volume's revert process would possibly fail, in order to prevent
  data loss, before the reverting process, we will create a snapshot
  for backup and then delete it if the operation succeed, end users can
  take advantage of that snapshot to recover their data.

- **Implement revert to snapshot in cinder drivers**.

  Add a function that vendors can use to overwrite the default
  ``revert_to_snapshot`` function with an optimized version. Vendors can
  implement the optimized version of the revert snapshot feature in their
  drivers::

      def revert_to_snapshot(self, context, volume, snapshot):
          """Is called to perform revert volume from snapshot.

          :param context: Our working context.
          :param volume: the volume to be reverted.
          :param snapshot: the snapshot data revert to volume.
          :return None
          """

  Exception will be caught and stored if any inner error raised from
  this function.

- **Add a generic revert to snapshot implementation function**.

  If the cinder driver can not support the revert to snapshot feature,
  the framework will use the generic way to implement the revert snapshot
  feature.

  1. create a temporary volume from snapshot (if the backend supports mount
     snapshot, we will directly mount the snapshot, and don't need to create
     and mount the temporary volume anymore).
  2. mount temporary volume to host.
  3. copy data from temporary volume or snapshot to original volume.
  4. delete the temporary volume.

Alternatives
------------

* There is an alternative that we can create a new volume from the snapshot
  (already implemented and used). But this has some drawbacks as we described
  in the 'Problem description' section.

* Implement snapshot revert allowing revert to any snapshot the volume has.

Data model impact
-----------------

New status 'reverting' to volume object and new status 'restoring' to
snapshot object.

REST API impact
---------------

Add a new revert action to volume action collections, the 'volume_id' and
'snapshot_id' are both required::

    URL = /v3/{tenant_id}/volumes/{volume_id}/action

    Method: POST

    Body = {
               'revert': {
                   'snapshot_id': <snapshot_id>
               }
           }

    Normal Response: Status code 202 with empty response body

    Error responses:

    1. 404, if the volume is not found.
    2. 409, if volume and snapshot's status are not 'available'or
       the sizes of volume and snapshot are not equal.
    3. 400, if the snapshot is not found or not belongs to the specified
       volume or is not the latest one.

Calling this method reverts a volume to the specified snapshot.
It is intended for both tenants and admins to use, and the policy.json
file will be updated to reflect allowed use by all.

Security impact
---------------

None

Notifications impact
--------------------

Notifications about the revert process(start, end, error) and
so does the related change notification will be add.

Cinder-client impact
--------------------
The cinder-client will add command to expose the revert API::

  usage: cinder revert-to-snapshot <snapshot>

  Revert a volume to the specified snapshot.

  snapshot: Name or ID of the snapshot to restore.

OpenStack-client impact
-----------------------

The openstack-client will add command to expose the revert API::

  usage: openstack volume snapshot restore <snapshot>

  Revert a volume to the specified snapshot.

  snapshot: Name or ID of the snapshot to restore.

Other end user impact
---------------------

This feature would also be implemented to cinder-ui.

Performance Impact
------------------

The in effect revert process might lock storage and consume a long time,
depending on the size of the volume.

Determining which snapshot is the latest requires a database query that
sorts by a timestamp (the created_at field on the snapshot object). This
would be slightly slower than a query that does not care about result
ordering.

Other deployer impact
---------------------

None

Developer impact
----------------

There will be one new driver entry point ``revert_to_snapshot``, driver
maintainers can implement optimised version of this functionality.
Also driver should pay attention to this cases.

1. During the reverting process, the specified snapshot can't be deleted
   (this means the snapshot should be recreated in the ``revert_to_snapshot``
   if that is deleted by the backend).

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  - zhongjun(jun.zhongjun2@gmail.com)
  - TommyLike(tommylikehu@gmail.com)

Work Items
----------

- Create APIs
- Create RPC calls
- Add generic revert_to_snapshot function.
- Add revert_to_snapshot interface and lvm's implementation
- Implement unit test and functional test
- Implement tempest test
- Implement cinder client
- Implement OSC
- Implement cinder UI
- Add API documents
- Update develop reference and support matrix

Dependencies
============

None

Testing
=======

Unit tests and manual testing, as well as tempest test for these
proposed APIs.

1. To guarantee that a volume restore actually took place, a new scenario
   test will be needed that writes data to a volume, creates a snapshot,
   modifies the volume, restores the snapshot, and ensures the original data
   is present in the volume.
2. To guarantee that this feature could work in the HA deploy mode.

Documentation Impact
====================

* API Ref: Add content about the API.
* Add revert to snapshot feature to develop reference.
* Add RevertToSnapshot capability to cinder support matrix.


References
==========

_`[1]`: https://specs.openstack.org/openstack/manila-specs/specs/ocata/manila-share-revert-to-snapshot.html
_`[2]`: https://wiki.openstack.org/wiki/CinderSupportMatrix
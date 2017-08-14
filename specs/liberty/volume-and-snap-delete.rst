..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Deletion of volumes with associated snapshots
=============================================

https://blueprints.launchpad.net/cinder/+spec/del-vols-with-snaps

Allow deletion of volumes with existing snapshots.  The proposal
is to integrate this with the existing volume delete path, using
an additional parameter to request deletion of snapshots as well
when deleting a volume.


Problem description
===================

When deleting a volume, the delete operation may fail due to snapshots
existing for that volume.  The caller is forced to examine snapshot
information and make many calls to remove snapshots, even if they are not
interested in the snapshots at all, and just want the volume gone.

Since snapshots are "children" of volumes in our model of the world, it
is reasonable to allow a volume and its snapshots to be removed in one
operation both for usability and performance reasons.

Use Cases
=========

* More friendly and expected behavior for end-users.

Currently, if a volume has snapshots, the basic user experience is:
    1. Try to delete volume
    2. Get back error message about it having snapshots
    3. Go delete X snapshots manually, become slightly frustrated
    4. Delete the volume


* Simpler for other software integrating with Cinder.

I received a request for this functionality because a project integrating
with Cinder would like to be able to just delete volumes without having to
handle logic for this.  I think that is a reasonable point of view (just as
it's reasonable to figure that a user shouldn't have to handle this).


* Faster and more efficient for some volume drivers.

There are two losses of performance in requiring back and forth between
cinder-volume and the backend to delete a volume and snapshots:
   A.  Extra time spent checking the status of X requests.
   B.  Time spent merging snapshot data into another snapshot or volume
       which is going to immediately be deleted.

This means that we currently force a "delete it all" operation to take
more I/O and time than it really needs to.  (The degree of which depends
on the particular backend.)


Proposed change
===============

A volume delete operation should handle this by default.

Phase 1:

This is the generic/"non-optimized" case which will work with any volume
driver.

When a volume delete request is received:
   1. Look for snapshots belonging to the volume, set them all to "deleting"
      status.
   2. Set the volume to "deleting" status.  (Done after #1 so as not to
      diverge more than needed from our current model of state transitions.)
   3. Issue a snapshot delete for each snapshot.
      (This loop happens in the volume manager.)
   4. If any snapshot delete operations fail, fail the operation and ensure
      the volume returns to available.  Any snapshots that were successfully
      deleted remain so.  Any snapshots which failed to be deleted are
      marked as 'error_deleting'.
      It may make sense to continue deleting snapshots if an error occurs,
      or it may be best to stop, depending on the type of error.  We
      probably need some experience with implementing this to be sure of
      the details for this.
   5. Volume manager now moves all snapshots in 'deleting' state to deleted.
      (volume_destroy/snapshot_destroy)

Phase 2:

This case is for volume drivers that wish to handle mass volume/snapshot
deletion in an optimized fashion.

When a volume delete request is received:
    Starting in the volume manager...
    1. Check for a driver capability of 'volume_with_snapshots_delete'.
       (Name TBD.)  This will be a new abc driver feature.
    2. If the driver supports this, call driver.delete_volume_and_snapshots().
       This will be passed the volume, and a list of all relevant
       snapshots.
    3. No exception thrown by the driver will indicate that everything
       was successfully deleted.  The driver may return information indicating
       that the volume itself is intact, but snapshot operations failed.
    4. Volume manager now moves all snapshots and the volume from 'deleting'
       to deleted.  (volume_destroy/snapshot_destroy)
    5. If an exception occurred, set the volume and all snapshots to
       'error_deleting'.  We don't have enough information to do anything
       else safely.
    6. The driver returns a list of dicts indicating the new statuses of
       the volume and each snapshot.  This allows handling cases where
       deletion of some things succeeded but the process did not complete.


Alternatives
------------

* Implement as the default behavior in volume delete.
  - Deemed not a suitable change at this time.

* Introduce this as a separate volume_action instead of in the standard volume
  delete path.
  - Does not help usability without client modifications.


Data model impact
-----------------

No direct impact.

In implementation, we need to ensure we don't end up with strange things
like a volume in a "deleting" status that has snapshots in "available"
status.  Thus, failures to delete a single snapshot in this model may
cascade to marking the volume and all other associated snapshots as
errored.  (Only relevant for phase 2 above. This doesn't happen if we
leave the snapshot and volume delete operations separate internally.)

REST API impact
---------------

Add a boolean parameter "delete_snapshots" to the delete volume
call, which defaults to false.

A volume delete with snapshots which previously returned 400 will now
succeed.

Security impact
---------------

None.

Notifications impact
--------------------

None.

All snapshot/volume delete notifications will still be fired.

Other end user impact
---------------------

New --delete-snapshots parameter for volume-delete in cinderclient.


Performance Impact
------------------

* Someone deleting a volume and all snapshots should be able to achieve
  this more quickly, and with fewer REST calls.

* Some storage backends will experience less load due to not having to
  merge snapshots being deleted.


Other deployer impact
---------------------

None.

Developer impact
----------------

* New, optional, driver interface:
    def delete_volume_and_snapshots(volume, snapshots[]):
       This should take whatever driver-specific steps are needed
       to delete the snapshots and associated volume data.

       The assumption can be made that any failed snapshot delete
       results in a failed volume, so this does not have to account
       for partial failures.

* Note: None of this has to happen at a level above the volume manager since
  the volume manager handles all related status updates.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eharney (spec, some implementation)

Other contributors:
  Other associates (implementation)

Work Items
----------

Investigation:

* Understand interaction w/ public/shared snapshots.


Implementation:

Rough order should be:
* Add parsing for new parameter to volume delete API
* Implement volume manager logic to delete everything
* Create an abc class for the new driver interface
* Implement volume manager logic to talk to the new driver interface
* Implement an optimized case for the LVM driver


Dependencies
============

None


Testing
=======

Tempest tests will be added to cover this.


Documentation Impact
====================

Need to document the new behavior of the volume delete call, as well
as related client examples, etc.


References
==========

* https://review.openstack.org/#/c/133822/
  This is not proposing the same thing as this spec!  It proposed to
  orphan the snapshots and transform them into volumes, or similar.

* https://bugs.launchpad.net/cinder/+bug/1276101
  Bug demonstrating one of the usability issues here


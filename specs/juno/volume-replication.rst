
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Volume Replication
==========================================

https://blueprints.launchpad.net/cinder/+spec/volume-replication

Volume replication is a key storage feature and a requirement for
features such as high-availability and disaster recovery of applications
running on top of OpenStack clouds.
This blueprint is an attempt to add initial support for volume replication
in Cinder, and is considered a first take which will include support for:
* Replicate volumes (primary to secondary approach)
* Promote a secondary to primary (and stop replication)
* Re-enable replication
* Test that replication is running properly

It is important to note that this is a first pass at volume replication.
The process of implementing replication for drivers has uncovered a
number of challenges that will be addressed in a future revision of
replication that will address the ability to have different replication
types and the ability to replicate across multiple backends.

While this blueprint focuses on volume replication, a related blueprint
focuses on consistency groups, and replication will be extended to
support it.

Problem description
===================

The main use of volume replication is resiliency in presence of failures.
Examples of possible failures are:

* Storage system failure
* Rack(s) level failure
* Datacenter level failure

Here we specifically exclude failures like media failures, disk failures, etc.
Such failures are typically addressed by local resiliency schemes.

Replication can be implemented in the following ways:

* Host-based - Requires Nova integration

* Storage-based

  - Typical block based approach - replication is specified between two
    existing volumes (or groups of volumes) on the controllers.
  - Typically file system based approach - a file
    (in Cinder context, the file representing a block device) placed in a
    directory (or group or fileset, etc) that is automatically copied to a
    specified remote location.

Assumptions:

* Replication should be transparent to the end-user, failover, failback
  and test will be executed by the cloud admin.
  However, to test that the application is working, the end-user may be
  involved, as they will be required to verify that his application is
  working with the volume replica.

* The storage admin will provide the setup and configuration to enable the
  actual replication between the storage systems. This could be performed
  at the storage back-end or storage driver level depending on the storage
  back-end. Specifically, storage drivers are expected to report with whom
  they can replicate and report this to the scheduler.

* The cloud admin will enable the replication feature through the use of
  volume types.

* The end-user will not be directly exposed to the replication feature.
  Selecting a volume-type will determine if the volume will be replicated,
  based on the actual extra-spec definition of the volume type (defined by
  the cloud admin).

* Quota management: quota are consumed as 2x as two volumes are
  created and the consumed space is doubled.
  We can re-examine this mechanism after we get comments from deployers.

Proposed change
===============

Introduction:

  The proposed design provides just a framework in Cinder for backend volume
  drivers to implement volume replication using the facilities in the storage
  backend.  As such, this spec provides guidance as to how volume replication
  should be implemented but the actual implementation will vary depending
  upon the backend in question.

  The key to enabling replication starts with adding an extra spec to the
  volume type to indicate that replication is desired.  That extra spec is
  then used in the volume driver to enable the set-up and control of
  replication on the storage backend in each of the different functions
  documented below.

  Since Cinder is just providing the framework for backend volume drivers
  to implement replication, details of replication implementation are left
  to the backend to implement.  The backend driver developer will need to
  decide for their storage backend the best way to enable replication.  For
  instance one storage provider may feel that implementing synchronous
  replication is the best choice while another storage provider may choose
  asynchronous.  A provider could also choose to make it a configurable
  option.  Implementing volume replication in Cinder in this manner allows
  the greatest flexibility to the backend developer to implement replication.

  It is also important to note that the developer documentation must provide
  examples of how this is implemented in the Storwize driver.  This is an
  important item to note as it is not currently possible to demonstrate
  volume replication in Cinder's reference implementation, LVM.  Therefore
  developer documentation will have to serve as the reference.

Add extra-specs in the volume type to indicate replication:

* capabilities:replication <is> True - if True, the volume is to be replicated,
  if supported, by the backend driver.  If the option is not specified or
  False, then replication is not enabled. This option is required to enable
  replication.

Create volume with replication enabled:

* Backend drivers that wish to enable replication will need to update their
  create_volume() function to check for the
  'capabilities:replication <is> True' extra spec.  It is up to the backend
  driver developers to implement replication in a manner that is compatible
  with their storage backend.

  When a replicated volume is created it is expected that the volume dictionary
  will be populated as follows:

  ** volume['replication_status'] = 'copying'
  ** volume['replication_extended_status'] = <driver specific value>
  ** volume['driver_data'] = <driver specific value>

  The replica volume is hidden from the end user as the end user will
  never need to directly interact with the replica volume.  Any interaction
  with the replica happens through the primary volume.

  Further details around the dictionary fields above may be seen in the data
  "Data Model Impact" section below.

Create Volume from Snapshot:

  If the volume type extra specs include 'capabilities:replication <is> True'
  for the new volume, the driver needs to create a volume replica at volume
  creation time and set up replication between the newly created volume and its
  associated replica.  The volume dictionary should be populated in the same
  manner as create volume.

Create Cloned Volume:

  If the volume type extra specs include 'capabilities:replication <is> True'
  for the new volume, the driver needs to create a volume replica at clone
  creation time and set up replication between the newly created volume and its
  associated replica.  The volume dictionary should be populated in the same
  manner as create volume.

Create Replica Test Volume:

  Create a clone of the replica (secondary) volume.  This clone can then be
  used for testing replication to ensure that fail-over can be executed when
  necessary.  It is important to note that this doesn't actually execute the
  the promote path as the intention is not to promote the replica but it gives
  a method to ensure that the replica contains data and would be useful if
  it had to be promoted.

  The administrator is able to access this functionality using the
  --source-replica option when creating a volume.

Delete volume:

  For volumes with replication enabled the replica needs to be deleted
  along with the primary copy.  So, if a volume type has
  'capabilities:replication <is> True' set, the driver will need to do the
  additional deletion.

Get Volume Stats:

  If the storage backend driver supports replication the following state should
  be reported:
  * replication = True (None or False disables replication)

Re-type volume:

  Changing volume-type is the mechanism an admin can use to make an existing
  volume replicated, or to disable replication for a volume.  Change the
  volume-type of a volume to a volume-type that includes
  'capabilities:replication: <is> True' (and didn't have it before) should
  result in adding a secondary copy to a volume.  Change the volume-type of
  a volume to a volume-type that no longer includes
  'capabilities:replication: <is> True' should result in removing the secondary
  copy while preserving the primary copy.

  Returns either:
    A boolean indicating whether the retype occurred, or
    A tuple (retyped, model_update) where retyped is a boolean
    indicating if the retype occurred, and the model_update includes
    changes for the volume db.

  The steps to implement this would look as follows:
  * Do a diff['extra_specs'] and see if 'replication' is included.
  * If replication was enabled for the original volume_type but is not
    not enabled for the new volume_type, then replication should be disabled.
  * The replica should be deleted.
  * The volume dictionary should be updated as follows:
  ** volume['replication_status'] = 'disabled'
  ** volume['replication_extended_status'] = None
  ** volume['driver_data'] = None
  * If replication was not enabled for the original volume_type but is
    enabled for the new volume_type, then replication should be enabled.
  * A volume replica should be created and the replication should
    be set up between the volume and the newly created replica.
  * The volume dictionary should be updated as follows:
  ** volume['replication_status'] = 'copying'
  ** volume['replication_extended_status'] = <driver specific value>
  ** volume['driver_data'] = <driver specific value>

Get Replication Status:

  This will be used to update the status of replication between the primary and
  secondary volume.

  This function is called by the "_update_replication_relationship_status"
  function in 'manager.py' and is the mechanism to update the status
  replication between the primary and secondary copies.

  The actual state of the replication, as the storage backed is aware of,
  should be returned and the Cinder database should be updated to reflect the
  status reported from the storage backend.

  It is expected that the following model update for the volume will
  happen:

  * volume['replication_status'] = <error | copying | active | active-stopped |
                                    inactive>
  **  'error' if an error occurred with replication.
  **  'copying' replication copying data to secondary (inconsistent)
  **  'active' replication copying data to secondary (consistent)
  **  'active-stopped' replication data copy on hold (consistent)
  **  'inactive' if replication data copy is stopped (inconsistent)
  * volume['replication_extended_status'] = <driver specific value>
  * volume['driver_data'] = <driver specific value>

  Note for get replication status, that the replication_extended_status and
  driver_data may not need to be updated.

Promote replica:

  Promotion of a replica means that the secondary volume will take over
  for the primary volume.  This can be thought of as a 'fail over' operation.
  Once promotion has happened replication between the two volumes, at the
  storage level, should be stopped, the replica should be available to be
  attached and the replication status should be changed to 'inactive' if the
  change is successful, otherwise it should be 'error'.

  A model update for the volume is returned.

  As with the functions above, the volume driver is expected to update the
  volume dictionary as follows:
  * volume['replication_status'] = <error | inactive>
  **  'error' if an error occurred with replication.
  **  'inactive' if replication data copy on hold (inconsistent)
  * volume['replication_extended_status'] = <driver specific value>
  * volume['driver_data'] = <driver specific value>

Re-enable replication:

  Re-enabling replication would be used to fix the replication between
  the primary and secondary volumes.  Replication would need to be
  re-enabled as part of the fail-back process to make the promoted
  volume and the old primary volume consistent again.

  The volume driver returns a model update to reflect the actions taken.

  The backend driver is expected to update the following volume dictionary
  entries:
  * volume['replication_status'] = <error | copying | active | active-stopped |
                                    inactive>
  **  'error' if an error occurred with replication.
  **  'copying' replication copying data to secondary (inconsistent)
  **  'active' replication copying data to secondary (consistent)
  **  'active-stopped' replication data copy on hold (consistent)
  **  'inactive' if replication data copy is stopped (inconsistent)
  * volume['replication_extended_status'] = <driver specific value>
  * volume['driver_data'] = <driver specific value>

Notes:

  The replication_extended_status should be used to store information that
  the backend driver will need to track replication status.  For instance,
  the Storwize driver, will use the replication_extended_status to track
  the primary copy status and synchronization status for the primary volume
  and the copy status, synchronization status and synchronization progress for
  the replica (secondary) volume.

  The driver_data field may be, optionally, used to contain any additional data
  that the backend driver may require.  Some backend drivers may not need to
  use the driver_data field.

Driver API:

* promote:  Promotes a replica that is in active or active-stopped state to
            be the primary.
* reenable: Reenables replication on a volume that is in inactive,
            active-stopped or error status.


Alternatives
------------

Replication can be performed outside of Cinder, and OpenStack can be
unaware of it. However, this requires vendor specific scripts, and
is not visible to the admin user, as only the storage system admin
will see the replica and the state of the replication.
Also all recovery actions (failover, failback) will require both the
the storage and cloud admins to work together.
While replication in Cinder reduces the role of the storage admin to
only the setup phase, and the cloud admin is responsible for failover
and failback with (typically) no need for intervention from the cloud
admin.

Data model impact
-----------------

* The volumes table will be updated:
** Add replication_status column (string) for indicating the status of
   replication for a give volume.  Possible values are:
*** 'copying' - Data is being copied between volumes, the secondary is
                inconsistent.
*** 'disabled' - Volume replication is disabled.
*** 'error' - Replication is in error state.
*** 'active' - Data is being copied to the secondary and the secondary is
               consistent.
*** 'active-stopped' - Data is not being copied to the secondary (on hold),
                       the secondary volume is consistent.
*** 'inactive' - Data is not being copied to the secondary, the secondary
                 copy is inconsistent.
** Add replication_extended_status column to contain details with regards
   to replication status of the primary and secondary volumes.
** Add replication_driver_data column to contain additional details that
   may be needed by a vendor's driver to implement replication on a backend.


State diagram for replication (status)

::

 <start>
                                          any error
                                          condition    +-------+
 Create volume   +-----+                +------------> | error |
                       |                               +---+---+
                       |                                   | Storage admin to
                       |                                   | fix, and status
                       |                                   | check will update
                 +-----+-----+                             |
 +-------------> |  copying  |           any state <-------+
 |               +-----+-----+
 |                    |
 |             status |
 |             check  |       status check
 |               +----+-----+ +------> +----------------+
 |               | active   |          | active-stopped |
 |               +----+-----+ <------+ +----------------+
 |                    |       status check
 |                    |
 |                    | promote to primary
 |                    |
 | re-enable     +----+-----+
 +------------+  | inactive |
                 +----------+

 <end>

REST API impact
---------------

Create volume API will have "source-replica" added:

{
    "volume":
    {
        "source-replica": "Volume uuid of primary to clone",
    }
}


* Promote volume to be the primary volume

  * Promote the secondary copy to be primary. the primary will become
    secondary and Replication should become inactive.
  * Method type: POST
  * Normal Response Code: 202
  * Expected error http response code(s)

    * 500: Replication is not enabled for volume
    * 500: Replication status for volume must be active or active-stopped,
      but current status is: <status>
    * 500: Volume status for volume must be available, but current status
      is: <status>

  * V2/<tenant id>/volumes/os-promote-replica/<volume uuid>
  * This API has no body


* Re-enable replication between the primary and secondary volume.

  * Re-enable the replication between the primary and secondary volume.
    Typically follows a promote operation on the replication.
  * Method type: POST
  * Normal Response Code: 202
  * Expected error http response code(s)

    * 500: Replication is not enabled
    * 500: Replication status for volume must be inactive, active-stopped,
      or error, but current status is: <status>

  * /v2/<tenant id>/volumes/os-reenable-replica/<volume uuid>
  * This API has no body

Security impact
---------------

* Does this change touch sensitive data such as tokens, keys, or user data?
  *No*.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
  *No*.

* Does this change involve cryptography or hashing?
  *No*.

* Does this change require the use of sudo or any elevated privileges?
  *No*.

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.
  *No*.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some
  examples of this include launching subprocesses for each connection, or
  entity expansion attacks in XML.
  *Yes*, enabling replication consume cloud and storage resources.

Notifications impact
--------------------

Will add notification for promoting and re-enabling replication for
volumes.

Other end user impact
---------------------

* End-user to use volume types to enable replication.

* Cloud admin to use the *replication-promote*, *replication-reenable* and
  *create --source-replica* commands in the python-cinderclient to execute
  failover, failback and test.

Performance Impact
------------------

* Extra db calls identifying if replication exists are added to retype,
  snapshot operations, etc will add a small latency to these functions.

Other deployer impact
---------------------

* Added options for volume types (see above)

* Add new driver capabilities, needs to be supported by the volume drivers,
  which may imply changes to the driver configuration options.

* This change will require explicit enablement (to be used by users)
  from the cloud administrator.

Developer impact
----------------

* Change to the driver API is noted above. Third party backends that wish
  to enable replication will need to add replication support to their driver.

* The API will expand to include consistency groups following the merge of
  consistency group support to Cinder.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ronenkat

Other contributors:
  Jay Bryant - E-Mail: jsbryant@us.ibm.com   IRC: jungleboyj

Work Items
----------

* Cinder public (admin) APIs for replication
* DB schema updates for replication
* Cinder driver API additions for replication
* Cinder manager update for replication
* Testing


Dependencies
============

* Related blueprints: Consistency groups
  https://blueprints.launchpad.net/cinder/+spec/consistency-groups

* LVM to support replication using DRBD, in a separate contribution.

Testing
=======

* Testing in gate is not supported due to the following considerations:

  * LVM has no replication support, to be addressed using DRBD in a separate
    contribution.
  * requires setting up at least two nodes using DRBD

* Should be discussed/addressed as support for LVM is added.

* 3rd party driver CI will be expected to test replication.

Documentation Impact
====================

* Public (admin) API changes.
* Details how replication is used by leveraging volume types.
* Driver docs explaining how replication is setup for each driver.
* Provide examples of volume replication implementation for
  the Storwize backend.

References
==========
Etherpad on improvements needed in documentation:
    https://etherpad.openstack.org/p/cinder-replication-redoc


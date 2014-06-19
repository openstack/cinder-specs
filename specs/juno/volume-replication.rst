
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
in Cinder, and is considered a first take, and will include support for:
* Replicate volumes (primary to secondary approach)
* Promote a secondary to primary (and stop replication)
* Synchronize replication with direction

This would further be enhanced in the future.

While this blueprint focuses on volume replication, a related blueprint
focuses on consistency groups, and replication would be extended to
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
  involved, as he will be required to verify that his application is
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
  created and the consumed space id double.
  We can re-examine this mechanism after we get comments from deployers.

Proposed change
===============

Each Cinder host will report replication capabilities:

* Replication_support: indicate if replication is enabled for this driver
  instance
* Replication_unit_id: device specific id used for replication
* Replication_partners: list of device specific ids that this node can
  replicate with
* Replication_rpo_range - supported RPO by this driver instance <min,max>
* replication_supported_methods - list of methods supported by the back-end

Add extra-specs in the volume type to indicate replication:

* Replication_enabled - if True, volume to be replicated if exists as extra
   specs. if option is not specified or False, then replication is not
   enabled. This option is required to enable replication.
* replica_same_az  - (optional) indicate if replica should be in the same AZ
* replica_volume_backend_name - (optional) specify back-end to be used as
  target
* replication_target_rpo - (optional) requested RPO (numeric, minutes) for
  the volume

Create volume with replication enabled:

* Scheduler selects two hosts for volume placement and sets up the replication
  DB entry
* Manager on primary creates the primary volume (as is done today)
* Manager on secondary creates the replica volume
* Manager on primary sets up the replication

Re-type volume:

* Replication_enabled: True->False:
  drop the replication and continue with the regular retype logic.
* Replication_enabled: False->True:
  after the retype logic selects back-ends (scheduler) and enables
  replication.

Promote to primary:

* Manager on secondary stops the replication.
* Switch between volume ids of primary and secondary
  (user sees no change in volume ids)

Sync replication:

* Manager on primary restarts the replication

Test:

* Create a clone of the secondary volume.

Delete volume:

* Disable the replication
* Delete secondary volume
* Delete primary volume (as is done today)

Cloning a volume:

* Since the replica are added after the primary is created, if we
  clone a volume and keep the volume-type, it will be replicated.

Snapshots:

* Snapshot for the primary volume works as is today, and create
  a snapshot on the primary. No snapshot is done for the replica.
* Snapshot for the replica (secondary) volume will fail.

Notes:

* Manager acts via the driver for back-end replication specific functions.
* Failover is "promote to primary" as described above.
* Failback is "sync replication" + "promote to primary".

Driver API:

* create_replica: to be run on secondary to create the volume
* enable_replica: to be run on primary to start replication
* disable_replica: to be run on primary, stops the replication
* delete_replica: to be run on secondary, deletes the replica target volume
* replication_status_check: to be run on all hosts, updating the replication
  status as observed from the back-end perspective
* promote_replica: to be run on secondary, make secondary the primary

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
and failback with (typically) not need for intervention from the clouds
admin.

Data model impact
-----------------

* A new replication relationship table will be created.
  (with its database migration support).

* On promote to primary, the ids of the primary and secondary volume entries
  will change (switch).

Replication relationship db table:

* id = Column(String(36), primary_key=True)
* deleted = Column(Boolean, default=False)
* primary_id = Column(String(36), ForeignKey('volumes.id'), nullable=False)
* secondary_id = Column(String(36), ForeignKey('volumes.id'), nullable=False)
* primary_replication_unit_id = Column(String(255))
* secondary_replication_unit_id = Column(String(255))
* status = Column(Enum('error', 'creating', 'copying', 'active', 'active-stopped',
                       'stopping', 'deleting', 'deleted', 'inactive',
                       name='replicationrelationship_status'))
* extended_status = Column(String(255))
* driver_data = Column(String(255))

State diagram for replication (status)::
 <start>
                                          any error
 Create replica   +----------+             condition   +-------+
 +--------------> | creating |          +------------> | error |
                  +----+-----+                         +---+---+
                       |                                   | Storage admin to
                       | enable replication                | fix, and status
                       |                                   | check will update
                  +----+-----+                             |
 +-------------> | copying  |           any state <--------+
 |               +----+-----+
 |                    |
 |             status |
 |             check  |       status check
 |               +----++----+ +------> +--+--+-+--------+
 |               | active   |          | active-stopped |
 |               +----++----+ <------+ +--+--+-+--------+
 |                    |       status check
 |                    |
 |                    | promote to primary
 |                    |
 |    sync       +----+--+--+
 +------------+  | inactive |
                 +-------+--+
 <end>

REST API impact
---------------

* Show replication relationship

  * Show information about a volume replication relationship.
  * Method type: GET
  * Normal Response Code: 200
  * Expected error http response code(s)

    * 404: replication relationship not found

  * /v2/<tenant id>/os-volume-replication/<replication uuid>
  * JSON schema definition for the response data::

     {
        'relationship':
        {
           'id': 'relationship id'
           'primary_id': 'primary volume uuid'
           'status': 'status of relationship'
           'links': '{ ... }'
        }
      }

* Show replication relationship with details

  * Show detailed information about a volume replication relationship.
  * Method type: GET
  * Normal Response Code: 200
  * Expected error http response code(s)

    * 404: replication relationship not found

  * /v2/<tenant id>/os-volume-replication/<replication uuid>/detail
  * JSON schema definition for the response data::

     {
        'relationship':
        {
           'id': 'relationship id'
           'primary_id': 'primary volume uuid'
           'secondary_id': 'secondary volume uuid'
           'status': 'status of relationship'
           'extended_status': 'extended status'
           'links': { ... }
        }
     }

* List replication relationship with details

  * List detailed information about a volume replication relationship.
  * Method type: GET
  * Normal Response Code: 200
  * Expected error http response code(s)

    * TBD

  * /v2/<tenant id>/os-volume-replication/detail
  * Parameters:

    *status*
       filter by replication relationship status
    *primary_id*
       Filter by primary volume id
    *secondary_id*
       Filter by secondary volume id

  * JSON schema definition for the response data::

     {
        'relationship':
        {
           'id': 'relationship id'
           'primary_id': 'primary volume uuid'
           'secondary_id': 'secondary volume uuid'
           'status': 'status of relationship'
           'extended_status': 'extended status'
           'links': { ... }
        }
     }

* Promote volume to be the primary volume

  * Switch between the uuids of the primary and secondary volumes, and
    make the secondary volume the primary volume.
  * Method type: PUT
  * Normal Response Code: 202
  * Expected error http response code(s)

    * 404: replication relationship not found

  * /v2/<tenant id>/os-volume-replication/<replication uuid>
  * JSON schema definition for the body data::

     {
        'relationship':
        {
           'promote': None
        }
     }

* Sync between the primary and secondary volume.

  * Resync the replication between the primary and secondary volume.
    Typically follows a promote operation on the replication.
  * Method type: PUT
  * Normal Response Code: 202
  * Expected error http response code(s)

    * 404: replication relationship not found

  * /v2/<tenant id>/os-volume-replication/<replication uuid>
  * JSON schema definition for the body data::

     {
        'relationship':
        {
           'sync': None
        }
     }

* Test replication by make a copy of the secondary volume available

  * Test the volume replication. Create a clone of the secondary volume
    and make it accessible, so the promote process can be tested.
  * Method type: POST
  * Normal Response Code: 202
  * Expected error http response code(s)

    * 404: replication relationship not found

  * /v2/<tenant id>/os-volume-replication/<replication uuid>/test
  * JSON schema definition for the response data::

     {
        'relationship':
        {
           'volume_id': 'volume id of the cloned secondary'
        }
     }

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

Will add notification for enabling replication, promoting, syncing and
dropping replication.

Other end user impact
---------------------

* End-user to use volume types to enable/disable replication.

* Cloud admin to use the *promote*, *sync* and *test* commands
  in the python-cinderclient to execute failover, failback and test.

Performance Impact
------------------

* Scheduler now needs to choose two hosts instead of one based on
  additional input from the driver and volume type.

* The periodic task will query the driver and back-end for status
  of all replicated volumes - running on the primary and secondary.

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

* Change to the driver API is noted above. Basically new functions are
  needed to support using replication.

* The API will expand to include consistency groups following merging
  consistency group support to Cinder.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ronenkat

Other contributors:
  None

Work Items
----------

* Cinder public (admin) APIs for replication
* DB schema for replication
* Cinder scheduler support for replication
* Cinder driver API additions for replication
* Cinder manager update for replication
* Testing

Note: Code is based on https://review.openstack.org/#/c/64026/ which was
submitted in the Icehouse development cycle.

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

References
==========

* Volume replication design session
  https://etherpad.openstack.org/p/juno-cinder-volume-replication


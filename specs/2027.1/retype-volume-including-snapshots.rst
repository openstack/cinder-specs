================================================
Retype volume with migration including snapshots
================================================

URL of launchpad blueprint:
[ https://blueprints.launchpad.net/cinder/+spec/retype-volume-including-snapshots ]

This specification proposes an enhancement to Cinder retype volume to allow
migrating an available or in-use volume *together with all of its snapshots*
between different backends with different volume types, preserving the lineage
of snapshots.

This specification focuses on the enhancement of the ``os-retype`` API. The
``os-migrate_volume`` API is out of scope.
The encrypted volume (regardless of source or destination) is excluded from
this specification.

Problem description
===================

Migrating volumes with snapshots is not supported.

[ https://docs.openstack.org/cinder/latest/admin/volume-migration.html ]

Use Cases
=========

* Datacenter Maintenance or Backend Upgrade: An operator needs to vacate all
  volumes from a storage backend that is being decommissioned or upgraded. Some
  volumes on that backend have snapshots (e.g., user-created restore points).
  Today, those volumes cannot be moved or retyped without deleting the
  snapshots, which is not acceptable to the users. With this feature, the
  operator could migrate such volumes (and their snapshots) seamlessly to a new
  backend with different volume types, preserving all restore points.

* Load Balancing and Tiering: Over time, an administrator might want to
  redistribute volumes across backends (for example, moving older,
  less-accessed volumes to a different storage tier). If a volume has snapshots
  (common for long-lived volumes), they currently block migration or changing
  volume type. Enabling snapshot-inclusive migration lets the admin move the
  volume to a new backend (perhaps a lower-cost tier) and change the volume type
  without losing the historical snapshots associated with it.

* Changing Backend Storage Vendor: A cloud vendor might want to change their
  backend storage vendor. If volume migration with snapshots capabilities is
  only supported by the storage driver, the cloud vendor will face storage
  vendor lock-in. Implementing this feature in software would allow them to
  introduce the best available storage whenever expansion is required, thereby
  preventing vendor lock-in, which is also why cross-backend volume movement is
  mediated by Cinder.

Proposed change
===============

We propose to enhance the Cinder retype volume flow so that a volume with
one or more snapshots can be migrated to another backend with different volume
types while preserving the Cinder snapshot objects and their relationship to
the volume.

Currently, retyping or migrating a volume including snapshots is difficult
because snapshots are created and managed by driver or storage-specific
mechanisms. A snapshot on one backend cannot generally be moved directly to
another backend in a backend-independent way.

This spec proposes a host-assisted snapshot migration mechanism. The basic idea
is to re-create each snapshot on the destination backend by using a temporary
volume created from the source snapshot. For each snapshot, Cinder creates a
temporary volume from the snapshot, migrates or copies that temporary volume to
the destination backend, and then creates the corresponding snapshot on the
destination backend. After all snapshots have been successfully re-created on
the destination backend, Cinder migrates the current volume to the destination
backend by using the existing volume migration mechanism.

Driver-assisted snapshot migration, where a driver or storage backend migrates
snapshots directly by using storage-native functionality, is out of scope for
this spec.

Key constraints and assumptions, based on the existing volume migration
behavior, are:

* Retyping an attached-volume requires Nova participation for the generic
  host-assisted copy path and is currently supported only by the Nova libvirt
  driver.
* Encrypted source volumes and encrypted destination volume types are not
  supported when the source volume has snapshots.
  Those requests are rejected during API pre-verification. When the volume has
  no snapshots, retype continues to follow the ordinary retype behavior.
* Retyping an in-use volume is driver dependent and is restricted for some VM
  states, for example ``SHUTOFF``, ``SUSPENDED``, and ``SOFT-DELETED``.
* The source driver supports creating a volume from a snapshot.
* The destination backend supports the required volume migration or copy
  operation for the temporary volumes used during snapshot migration.
* Source backend snapshot artifacts remain intact until the volume retyping
  cutover succeeds. The preserved Cinder snapshot records may be repointed
  during the phase-1 finalization described below, with compensating rollback
  for caught failures. The "in-use" source volume remains writable throughout
  the retyping process.

Workflow details
----------------

Request for the retype API
~~~~~~~~~~~~~~~~~~~~~~~~~~

When a volume migration between two backends with different volume types is
requested, Cinder determines whether the source volume has snapshots.

Pre-verification
~~~~~~~~~~~~~~~~

If the volume has snapshots, Cinder verifies that the source and destination
backends satisfy the requirements for host-assisted snapshot migration. This
includes the following items.

* The normal Cinder destination volume-type quota reservation for the
  preserved volume and its snapshots succeeds. This includes snapshot count
  and aggregate capacity where snapshot-gigabyte quota is enabled.
* The source backend may need headroom to materialize one temporary source
  volume from a snapshot. The capacity consumed by this operation is
  driver-dependent; a backend can use a space-efficient clone when it supports
  one. Cinder processes snapshots sequentially, so headroom for all temporary
  source volumes at once is not required. This version does not define a
  generic driver or scheduler capacity contract for this transient
  requirement; capacity exhaustion is handled as an execution failure rather
  than a synchronous API preflight result.
* For capacity planning, a destination backend without space-efficient
  snapshots may temporarily need up to ``(number of snapshots + 1)`` times the
  source volume size. This is a worst-case estimate, not a flat requirement;
  COW/ROW cloning can reduce it when the destination backend supports it.
* Temporary source volumes, the staging destination volume, and staging
  destination snapshots are internal migration resources. They are created
  through the Cinder internal-resource path with ``use_quota=False`` and do not
  consume Cinder quota from the requesting project or a service project.
* The volume status is in-use or available.
* All snapshots status are available.
* The source volume is not encrypted and the destination volume type is not
  encrypted.
* The source and destination backends must advertise the
  ``snapshot_migration_support`` capability. The scheduler filters destination
  candidates and the volume manager verifies both backends before creating
  migration artifacts.
* The number of snapshots for the volume must not exceed the configured limit
  for snapshot-aware retype operations. A value of -1 indicates no limit, while
  0 disables snapshot-aware retype requests.

Migrate snapshots
~~~~~~~~~~~~~~~~~

If the source and destination backends satisfy the requirements, Cinder migrates
all snapshots as follows:

* Create a staging destination volume on the target backend with the target
  volume type. This volume is reused to rebuild the snapshot chain and later
  becomes the migrated volume.
* The system enumerates all snapshots for the volume, ordered oldest -> newest.
* For each snapshot:

  * Materialize the source snapshot into a temporary source volume by using
    the source driver's existing ``create_volume_from_snapshot`` capability.
  * Transfer the point-in-time data from the temporary source volume into the
    staging destination volume. Drivers may optionally optimize this data
    copy, but the baseline flow uses the existing generic host-assisted
    copy path.
  * After the copy completes, create a normal snapshot of the staging
    destination volume on the destination backend. This flow relies on copying
    data into the staging destination volume first and then invoking the
    backend's ordinary snapshot operation.
  * Delete the temporary source volume before processing the next snapshot.
  * Treat the newly created destination snapshot as a staging artifact for the
    original source snapshot. During finalization, Cinder preserves the
    original Cinder snapshot object by switching its backend/provider fields to
    refer to the destination snapshot artifact, and drivers may apply
    backend-specific adopt or rename logic if needed.

* Maintain and/or reconstruct snapshot lineage:

  * The default rebuild order is oldest -> newest by ``created_at``. A driver
    that exposes parent/child or delta dependencies may override
    ``get_snapshots_in_migration_order(snapshots)`` to return a dependency-safe
    order.
  * The generic path does not promise to preserve backend-native lineage that
    is not exposed to Cinder; it rebuilds each destination snapshot from the
    staging volume. Drivers that require native lineage must advertise and
    implement the appropriate ordering and adoption behavior.
  * Backend-native snapshots or reference points that have no Cinder snapshot
    record are out of scope. They are not enumerated or rebuilt, may not
    survive the retype, and may still be attached to the source volume when
    the migration deletes it.
  * Ensure the destination snapshot set can be used for clone/restore
    operations as with the source.

Since all snapshots on the destination backend are created sequentially from the
same volume, COW or ROW cloning can be leveraged if the destination backend
supports it.

Migrate volume
~~~~~~~~~~~~~~

After all snapshots have been successfully migrated, the base volume data will
be transferred to the destination volume, which will then become the canonical
volume.

* If the volume status is ``available``, invoke the existing volume migration.
* If the volume status is ``in-use``, the base volume data transfer uses a
  host-assisted strategy with the same Nova-assisted copy mechanism used by the
  existing attached-volume migration flow where applicable.

  * Attach the destination volume to the same instance/host where the source
    volume is attached (without changing the guest-visible device until the
    final swap/commit step).
  * Invoke the existing in-use migration data copy flow where Cinder calls Nova
    to copy the data between the source and destination volumes.

  * Complete the migration by updating Cinder state to treat the destination
    volume as the canonical volume on the new backend.

Cinder DB update
~~~~~~~~~~~~~~~~

After the base volume data has been successfully transferred, Cinder updates
the volume and snapshot backend information so that the volume and all of its
snapshots refer to the destination backend. The snapshot-to-volume relationship
exposed through the Cinder API is preserved.

* Update the ``host`` / backend fields for the original volume record.
* Update ``provider_location``, ``provider_id``, and ``provider_auth`` fields
  for the original snapshot records so that existing Cinder snapshot
  records refer to the destination backend snapshot artifacts.
* Update the snapshots' ``volume_type_id``.
* Ensure snapshot records continue to reference the retyped volume.
* Finalize the snapshot records with an application-level two-phase protocol,
  rather than assuming that all per-snapshot database saves are one database
  transaction:

  * In phase 1, for each staging snapshot, the destination driver's
    ``update_migrated_snapshot(ctxt, snapshot, new_snapshot)`` hook returns
    the provider fields and optional metadata for the adopted artifact. The
    preserved source Cinder snapshot row, including its original ID, is then
    repointed to those fields and saved. The temporary destination snapshot
    row is removed, while the original source provider fields are retained as
    cleanup handles.
  * If any phase-1 update fails, Cinder compensates by restoring every
    already-updated snapshot row to its source provider fields and removes the
    adopted destination artifacts where possible. The source volume and
    snapshots remain usable.
  * In phase 2, after the volume provider update and the atomic swap of the two
    volume records succeed, Cinder deletes the retained source snapshot
    artifacts. Cleanup is best effort; failures are logged with Cinder
    resource IDs and ``provider_location`` where available for manual
    recovery.

  A process failure during phase 1 can leave the volume in ``completing`` with
  a mix of adopted and pending snapshots. Cinder does not automatically resume
  or unwind that window; it records the breakdown in the cinder-volume log for
  operator inspection.

Essentially, after the volume retyping, from an end-user perspective the volume
and its snapshots should function exactly as before, just now residing on a
different backend.

Failure handling
----------------

* Synchronous API validation errors are returned before the asynchronous
  migration starts. Invalid input such as an unsupported microversion,
  ``max_snapshots_per_migration`` being zero, a snapshot-count limit being
  exceeded, or an encryption restriction returns ``400 Bad Request``. A
  transitional or error-state snapshot returns ``409 Conflict``, and a
  destination volume-type quota reservation failure returns ``413 Request
  Entity Too Large``. Existing ordinary retype validation keeps its existing
  error behavior.
* Backend capability and transient-capacity failures are detected by the
  scheduler or volume manager after the request is accepted. They fail the
  asynchronous migration and set the volume migration status to ``error``;
  they are not represented as synchronous capacity guarantees.
* If any operation fails during snapshot migration, base-volume data
  migration, or finalization, Cinder attempts to roll back destination
  artifacts. Source backend snapshot artifacts remain intact through phase 1.
  Snapshot records repointed during phase 1 are restored if Cinder catches a
  failure, while a process interruption in that phase requires the manual
  recovery described below.
* On restart or failover, an interrupted, unattached ``migrating``
  snapshot-aware migration is recovered through the existing cleanup worker:
  Cinder attempts to delete staged destination snapshots, the destination
  staging volume, and any per-snapshot temporary source volumes, then marks
  the source migration as an error. An attached ``migrating`` migration, where
  Nova may already have pivoted the guest, and any interrupted ``completing``
  migration are not automatically rolled back.
* A volume left in ``retyping`` without a persisted migration phase is also
  left unchanged because Cinder cannot determine whether a native driver
  retype already mutated the backend. Cleanup failures leave source data
  intact, are logged with resource identifiers and provider information where
  available, and require administrator inspection and manual recovery.

Alternatives
============

Keep the REST API unchanged and use a cinder.conf option to enable the migration
of volumes with snapshots. In this case, ``migration_policy = on-demand`` would
retain its current API structure, while the deployment configuration would
determine whether snapshots are included. This means the same API request may
produce different results based on the deployment, with no way for the caller to
know which behavior to expect. Moreover, it denies the API user the ability to
opt into a potentially costly operation. This alternative is rejected. The
deployer control is instead a bound provided by
``max_snapshots_per_migration`` (``-1`` for no limit, ``0`` to disable, and a
positive value to cap the number).

Users manually delete snapshots, then retype the volume (not acceptable in most
production environments).

Users perform an application-level data migration (create a new volume on the
new backend and copy data within the guest), then recreate snapshots. This is
operationally heavy and does not preserve snapshot lineage/IDs.

Data model impact
=================

No new database schema is required. The existing volume
``migration_status`` values, workers table, temporary resource records, and
internal-resource markers are used to recover interrupted work. No
per-snapshot migration status field is added.

REST API impact
===============

Add an ``include_snapshots`` boolean option to the ``os-retype`` request in a
new microversion of the ``POST /v3/{project_id}/volumes/{volume_id}/action``
(Retype a Volume) API. This option requires the new
``volume:retype_include_snapshots`` policy, whose default rule is administrator
only. The existing ``volume:retype`` policy remains required for all retype
requests.

Requests that use this value with an older microversion fail with a
``400 Bad Request`` error.

The request is accepted only when the existing retype preconditions are met,
including that the volume status is ``available`` or ``in-use``, the volume is
not part of an active migration, the requested volume type differs from the
current volume type, the volume is not in a consistency group or group, and any
existing multiattach restrictions are satisfied. When the volume has snapshots,
all snapshots must be in ``available`` status. A snapshot in a transitional or
error status causes a ``409 Conflict`` error. The snapshot count and encryption
prechecks raise ``400 Bad Request``: a configured value of
``max_snapshots_per_migration=0`` disables snapshot-aware retype, a positive
value rejects a volume whose snapshot count exceeds the limit, and encrypted
source volumes or encrypted destination volume types are rejected when
snapshots are included. The normal destination volume-type quota reservation
is also performed before scheduling; a quota reservation failure returns
``413 Request Entity Too Large``. Capacity for transient resources is not
preflighted generically; capacity exhaustion is an asynchronous migration
failure handled by rollback.
Request Example:

.. code-block:: json

   {
        "os-retype": {
                "new_type": "new-storage-backend",
                "migration_policy": "on-demand",
                "include_snapshots": true
        }
   }

In this model, ``migration_policy`` continues to govern whether migration can
occur, and ``include_snapshots`` determines if snapshots are included when
migration is necessary. The default for ``include_snapshots`` is false,
maintaining existing API behavior.

Specifically, ``migration_policy = never`` with ``include_snapshots = true``
will not allow migration. A retype that can be completed without migration may
still succeed, whereas one that requires migration will fail, consistent with
current behavior.

While the retype API is being executed, operations on the volume and its
snapshots initiated by other API requests will be blocked. A ``409 Conflict``
error will be returned for blocked API requests.

Security impact
===============

The ``include_snapshots`` option introduces a new administrator-only operation.
Cinder enforces the ``volume:retype_include_snapshots`` policy in addition to the
existing ``volume:retype`` policy when the option is requested.

Active/Active HA impact
=======================

The retyping snapshot+volume workflow requires careful coordination.

* The existing retype locks/coordination mechanisms should be reused.
* The volume's ``migration_status`` flag will be used to coordinate and prevent
  concurrent operations on the retyping volume.

Upgrade impact
==============

The volume RPC API is bumped to version ``3.22`` for the
``include_snapshots`` argument to ``retype``. The snapshot finalization RPCs
``update_migrated_snapshot`` and ``delete_migrated_source_snapshot`` require
RPC version ``3.21``. Cinder selects its volume RPC version cap from the
minimum version advertised by all registered cinder-volume services. During a
rolling upgrade, the API accepts a snapshot-aware retype request, but the
scheduler fails it cleanly if any registered cinder-volume service cannot
receive version ``3.22``. The scheduler releases the quota reservations and
restores the volume status to its pre-request ``available`` or ``in-use``
value; ``migration_status`` remains at its pre-request value, normally
``None``. Once all registered cinder-volume services support version ``3.22``,
the version ``3.21`` finalization requirement is also satisfied. Ordinary
retypes continue to use the existing compatibility path.

Notifications impact
====================

None initially.

Optionally, a follow-up improvement could add progress notifications for each
snapshot and for the overall workflow.

Other end user impact
=====================

* Users benefit from retained snapshots after a retype operation that requires
  migration.
* This feature is limited to retype-with-migration flows. The non-retype Cinder
  migrate API is not changed and will not support migrating volumes with
  snapshots as part of this spec.
* No changes required to user commands for snapshot management.

Performance Impact
==================

Retyping volume duration increases with:

* number of snapshots,
* snapshot sizes / copy mechanisms,
* and volume size.

overall time becomes:

* (vendor volume migration time) + (snapshot migration time), where snapshot
  migration cost is roughly proportional to volume size multiplied by the
  number of snapshot generations.

For example, with a 2 TiB volume and 10 snapshots, the generic path may copy
approximately 22 TiB: 20 TiB for the ten snapshot generations plus 2 TiB for
the current volume. At a sustained effective throughput of 100 MiB/s, that is
about 64 hours of data transfer before protocol and backend overhead. The
destination may also need up to approximately 22 TiB of transient space in
the worst case without space-efficient snapshots, while sequential processing
limits temporary source materialization to about one volume at a time. Sparse
copies and COW/ROW snapshots can reduce both figures; operators should size
capacity and maintenance windows for the worst case rather than this example.

Other deployer impact
=====================

* Operators should be aware that retyping a volume may take longer when the
  volume includes many snapshots.
* Backends should support necessary snapshot operations for best results.
* Operators can set ``max_snapshots_per_migration`` in ``cinder.conf``:
  ``-1`` permits any number of snapshots, ``0`` disables snapshot-aware
  retype, and a positive value limits the number accepted for one retype. The
  default is ``-1``.

Developer impact
================

Drivers that support snapshot-aware migration must advertise the
``snapshot_migration_support`` capability on source and destination backend
stats. They may optionally implement these hooks:

* ``copy_volume_data_to_existing_volume(context, src_vol, dest_vol,
  remote=None) -> bool`` to copy data into the existing staging destination.
  Returning ``True`` means the driver completed the copy; returning ``False``
  selects the generic host-assisted copy path.
* ``update_migrated_snapshot(ctxt, snapshot, new_snapshot) -> dict | None`` to
  adopt a rebuilt snapshot and optionally return any corrected
  ``provider_location``, ``provider_id``, ``provider_auth``, and ``metadata``
  values. The mapping may be partial; omitted fields, an empty mapping, or
  ``None`` use the staging snapshot's provider fields and merged metadata.
* ``get_snapshots_in_migration_order(snapshots) -> list`` when backend lineage
  requires an order other than the default oldest-first order.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
 Fuli Qi <fqi>

Work Items
----------

1. Extend the existing retype, scheduler, and volume-manager workflow to
   orchestrate:

   a. snapshot enumeration,
   b. snapshot migration loop,
   c. volume migration,
   d. metadata update,
   e. rollback/cleanup tasks.

2. Implement helpers for snapshot migration.
3. Add driver capability checks and fallbacks.
4. Add unit tests for workflow decisions and failure paths.
5. Add functional tests with at least two backends.
6. Documentation update.

Dependencies
============

No new external dependencies.
This feature builds on top of existing Cinder and OpenStack services.
It will use the Cinder volume manager as already integrated.

Testing
=======

* Unit tests for:

  * ordering of snapshot migration,
  * rollback logic,
  * metadata update correctness,
  * in-use constraints and VM state checks,
  * API preflight decision coverage for snapshot-aware retype requests,
    including acceptance and rejection paths for snapshot presence,
    transitional snapshot state, volume status, migration policy,
    encryption-related constraints, and other existing retype preconditions.

* Functional tests for:

  * successful snapshot-aware retype flows for volumes that include N snapshots
    across two backends with different volume types.
  * API preflight validation scenarios for snapshot-aware retype requests,
    covering representative acceptance and rejection cases before asynchronous
    execution begins.
  * representative validation cases should include existing retype
    preconditions and snapshot-aware specific checks such as transitional
    snapshots, snapshot-count limits, and encryption-related constraints.
  * asynchronous failure when destination capacity is insufficient, including
    cleanup of temporary resources and preservation of source snapshots.
  * restart or failover during snapshot migration, including automatic cleanup
    for an interrupted, unattached ``migrating`` operation and operator
    recovery for an attached ``migrating``, an interrupted ``completing``, or
    an ambiguous ``retyping`` operation.

Documentation Impact
====================

* Update Cinder admin documentation to describe behavior and caveats.
* Document driver and hypervisor constraints (e.g., libvirt requirement for
  generic attached-volume migration).
* Document the current encryption limitation for snapshot-aware retype: when
  snapshots are present, encrypted source volumes and encrypted destination
  volume types are rejected during API validation.
* Document that snapshot-heavy volumes will take longer to retype.
* Document ``max_snapshots_per_migration`` in ``cinder.conf``: ``-1`` means
  unlimited, ``0`` disables snapshot-aware retype, and a positive value caps
  the number of snapshots accepted in one operation.
* Document that generic destination-capacity preflight is not available in this
  version. Operators must size the destination for the worst-case temporary
  copy, and capacity exhaustion is reported as an asynchronous migration
  failure.
* Document recovery of interrupted operations: inspect cinder-volume and Nova
  logs, attachment state, Cinder resource records, and backend artifacts.
  Allow the cleanup worker to handle an interrupted, unattached ``migrating``
  operation. Manually recover an attached ``migrating``, an interrupted
  ``completing``, or an ambiguous ``retyping`` operation, and remove orphaned
  artifacts only after confirming which side is canonical.

References
==========

None.

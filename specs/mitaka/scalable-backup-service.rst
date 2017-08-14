..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Scalable Backup Service
=======================

https://blueprints.launchpad.net/cinder/+spec/scalable-backup-service

Because cinder backup workloads run as a dedicated service, they have
potential to be run independently of other cinder volume services.
Ideally, backup services would be fired up on demand in the cloud as
backup workloads are generated, and elastically de-commissioned as
workloads go away.  Ideally, backup tasks would farmed out to the
least busy of the available backup services.

Pursuit of this ideal is blocked today by a tight coupling of cinder
backup service to cinder volume service.

This spec is dedicated to breaking this tight coupling of the cinder
backup and volume services.

That is the first step in working towards a truly elastic,
horizontally scalable backup service.  When this coupling is loosened,
multiple backup services can be fired up concurrently and run without
any requirement that they they be colocated with the volume services
or with one another for that matter.

We expect that there will be followup work to address subsequent steps
such as scheduler support for backup tasks and for dynamic generation
of backup service processes using VMs or containers rather than
dedicated physical nodes, but those subjects would be addressed by
followup specs and are not in scope here.

One thing at a time.

Problem description
===================

When the backup service backs up or restores a volume, it uses a
*backup driver* specific to a chosen and configured backup repository
to interact with e.g. a swift or Ceph object store, an NFS, glusterfs,
or generic POSIX filesystem, or Tivoli Storage Manager. This backup
driver plugs in to the backup manager and provides functionality to
put data in the backup repository or to retrieve it from the
repository.  A particular backup service process has only one backup
driver and therefore only one backup repository.  It may, however,
perform backup and restore operations for volumes for *any* of the
*enabled backends* handled by the volume service.

The backup service requires backend-specific functionality to attach
to any particular volume in order to read it for backup or to write to
it for restore.

Today, this backend-specific information is provided as follows.  When
the backup service process starts up, the backup manager loads the
volume manager module for each enabled backend configured in
``cinder.conf``.  When an OpenStack tenant or administrator makes a
request to create a backup from a cinder volume or to restore a backup
to a cinder volume, the backup api looks at the ``host`` field for the
*volume* in question and directs the rpc for the backup or restore
operation to the backup manager on that node.  There, the backup
manager selects the ``volume`` manager for the relevant volume backend
and invokes its *driver's* ``backup_volume`` or ``restore_volume``
method, passing that method the backup *object* and the backup
*driver* (swift, ceph, NFS, etc.) for the configured backup
repository.  Then the *volume driver* attaches to the volume in
question, opens it, and invokes the backup driver's ``backup`` or
``restore`` method using the file handle from the *volume driver's*
attach and open operation.

Thus today the backup manager does not invoke its backup driver
methods directly; it finds an appropriate volume driver via the
appropriate volume manager, and the volume driver invokes the
appropriate backup driver.  The volume driver by definition runs on
the cinder node that runs the volume service for the relevant backend,
so since the backup manager, volume driver, and backup driver code all
run in the context of a single operating system process on a single
node, the backup service is necessarily tied to the node that also
runs the volume service.

When multiple pools and backends run on the same node, this means that
concurrent backups end up running into the resource constraints of a
single node.  Since backup tasks do not flow through a scheduler, they
typically also run with the resource constraints of a single operating
system process.

Use Cases
=========

* An OpenStack tenant or administrator needs to do backups on a
  schedule or restores on demand such that to get the work done in a
  timely manner many backups and restores need to run concurrently.

* Resource requirements to handle peak backup and restore loads exceed
  the capabilities of a standard physical node.

Note that the need for concurrent backup and restore operations,
especially to the same backend, may be artificially suppressed
historically because of the lack of support for *live* backups.  We
anticipate that now that backups of *in-use* volumes are supported
([3] [5]) this need will significantly increase.

Proposed change
===============

Leverage remote attach capability to move the functionality provided
by the *volume* driver into the backup manager.  That way the backup
manager can invoke the backup driver directly rather than having to
load the volume manager at startup and run the relevant volume
manager's driver.

Nowadays attaching a volume is done using a ``brick`` library that
builds an appropriate ``connector`` for the attachment given
information about the backend (iSCSI, FibreChannel, NFS, whatever)
obtained by running a volume driver ``initialize_connection`` method
-- either directly or via an rpc to the appropriate volume service.
In the case of a *remote* attachment, ``initialize_connection`` is
invoked by rpc.  This is the only part of the workflow that actually
requires the volume service, and since it can be an rpc invocation,
this means that the direct load of volume manager and volume driver
can be eliminated in the backup service itself and the backup service
can therefore run on a different node than the node for the backend's
volume service.

Now that backups of *in-use* volumes are supported, volume drivers
can supply an ``attach_snapshot`` method, which is then used as
optimization instead of attaching a temporary volume copy of the
source volume.  In the initial implementation [6], an ``attach_snapshot``
method was added that only allows for local attaches and the reference
``lvm driver`` explicitly uses ``local_path`` when getting volumes
for backup operations [5].  As part of the work implementing this
blueprint spec, we will need to revisit snapshot attachment to allow
for remote attaches.  Drivers like ``lvm driver`` that cannot support
remote attach for snapshots will need to fall back to using temporary
volumes instead [5].
* Create a temporary volume from the original volume.
* Backup the temporary volume.
* Clean up the temporary volume.


Alternatives
------------

 * Keep the existing scheme.  Backup service continues to work but
   cannot scale out.

   The node running backup service has to be scaled up to handle
   projected peak backup workload and is likely to be either
   underutilized or underpowered for most actual backup workloads.

 * Instead of loading the volume manager directly from the backup
   manager, expose ``backup_volume`` and ``restore_volume`` methods in
   the volume manager, have these invoke the corresponding volume
   driver's methods by the same name, and have the backup manager use
   rpc to the volume service to trigger the volume manager's
   ``backup_volume`` and ``restore_volume`` methods.

  This approach would reduce memory requirements for the backup
  service process since it would no longer load the volume managers
  for all enabled backends. And the linkage between backup manager and
  volume service is now loosely coupled.  However, the bulk of the
  work - data transfer, encryption, compression - is done by the
  backup driver, which still remains tightly coupled to the volume
  service.


What this specification does not solve
----------------------------------------

* A backup task scheduler.

  The backup api process can get a list of active backup services from
  the database and choose as an rpc destination e.g. the first
  service, make a random choice, or round-robin among the choices.
  This is where a call to a scheduler could go in the future, but a
  scheduler is not itself in scope for this spec.

* Elastic backup service placement.

  Backup services will be started more or less manually by an
  administrator or configured to start on boot of a node.  One can
  imagine a dynamic mechanism for starting service VMs or containers
  triggered by the backup api as workloads arrive.  The decoupling of
  backup and volume services addressed by this spec is a pre-condition
  for such an elastic backup service placement capability but it is
  only a small step towards enabling such a capability.

Service ``init_host`` cleanup
-----------------------------

At startup, the current backup service code makes an attempt to
discover and cleanup orphaned, incomplete backup and restore
operations (e.g., they were in process when the backup process itself
was terminated).  The backup service assumes that it is the *only*
backup process, so that if it finds backups in creating or restoring
state at startup it can safely reset their state and detach the
volumes that were being backed up or restored.

This assumption is not safe if multiple backup processes can run
concurrently, and on separate nodes.  At startup, a backup service
needs to distinguish between in-flight operations that are owned by
another backup-service instance and orphaned operations.

Eventually, it will make sense for a backup service process to
cleanup stuff left behind either by earlier incarnations of itself
or by other abnormally terminated backup processes.  A solution to
this general problem, however, requires a reliable capability
to auto-fence oneself on connection loss as being developed as
part of the solution for Active-Active HA for the cinder volume
service [7].


Here we will align with the community decision at the Mitaka design
summit to defer the auto-fencing capability and start on Active-Active
HA for the cinder volume service without automatic cleanup, by
restricting backup service initialization cleanup to leftovers from
the same backup service.

The ``host`` field for a backup object will be set to the host for the
backup service to which the backup operation is cast. The status update
of the backup and host update will be handled in an transaction. Cleanup
at initialization can then be restricted to leftover objects that
chain through their corresponding backup object to a ``host`` field
matching oneself. Compare-and-swap DB operation will be used to prevent
race conditions.

For an example of how the ``host`` field will be set, consider
a volume with a backend handled by volume service on node A where
backup service processes are running on node B and node C.  When
a backup is created using the service on node B, the ``host`` field
for the backup object will be set to B.  When restoring from that
backup using the backup service on node C, the ``host`` field for
the backup object will be set to C.

Cleanup of associated volumes, temporary volumes, and temporary
snapshots will be done via rpc to the appropriate volume service host.

Note that the backup object contains a ``volume_id`` field for the
volume it backs up, as well as ``temp_volume_id`` and
``temp_snapshot_id`` fields for live backups, but it does not
currently keep the id of volumes to which it is restoring backups.  We
will need to add this field in order to determine orphaned
restore-operation volumes.

Special Volume Driver Backup/Restore Considerations
---------------------------------------------------

Since the functionality of the current volume driver ``backup_volume``
and ``restore_backup`` methods will in this proposal move into the
backup manager, these methods will no longer be needed and can be
removed from the codebase.  That said, some volume drivers override
these with methods that apparently have a bit more "special sauce"
than just preparing their volume for presentation as a block device.

We will need to analyze the codebase to root out any of these and
determine how to accommodate any special needs.

An example is the vmware volume driver [4], where a "backing" and
temporary vmdk file are created for the cinder volume and the
temporary vmdk file is used as the backup source.  We will have to
determine whether all this can be done in the volume driver's
``initialize_connection`` method during ``attach``, or whether we will
require an additional rpc hook to a *prepare_backup_volume* method or
some such for volume drivers of this sort.


Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

TBD.

We need to understand exactly where it is necessary to elevate
privileges in running backup and restore operations and ensure that
there is no unnecessary elevation above the normal privileges for the
admin or tentant requesting the these operations.


Notifications impact
--------------------

We should be able to do exactly the same backup service notifications
as those done now.

Other end user impact
---------------------

No change in function or client interaction.

Performance Impact
------------------

* Backup process will be lighter weight since volume manager and
  drivers are no longer loaded.

* The proposed change enables running multiple backup processes as
  required.

Other deployer impact
---------------------

Backup service can now run on multiple nodes and no longer has to run
on the same node as the volume service handling a volume's backend.

Developer impact
----------------

Enables potentially valuable future features such as backup scheduler
or elastic backup service placement.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Tom Barron (tbarron, tpb@dyncloud.net)

Other contributors:
* LisaLi (lixiaoy11, xiaoyan.li@intel.com)
* Huang Zhiteng (winston-d, winston.d@gmail.com)

Work Items
----------

* Write the code.  A POC is available now [1].

* Determine and address any impact on existing volume drivers that
  have their own backup or restore methods.

* Run/test the new code with multiple backup processes running on
  multiple nodes, other than the node or nodes where the volume
  services run for enabled backends.

Dependencies
============

None

Testing
=======

* Unit tests will be extended to cover new backup code for
  functionality formerly provided by volume driver.

* Unit tests for backup manager and volume drivers will be modified to
  reflect code removed from the backup service to load volume manager,
  run volume drivers, etc. and from the volume driver to run backup
  and restore operations.

* Existing tempest tests should provide sufficient coverage to ensure
  that current functionality does not regress.  Potentially new
  multi-node tempest tests could be added to verify distributed
  interactions.  We should take advantage of opportunities to extend
  current tempest coverage for backup and add functional tests for
  backup when this is feasible.


Documentation Impact
====================

Update with new deployment options.

References
==========

* [1]: https://review.openstack.org/#/c/203291
* [2]: https://etherpad.openstack.org/p/cinder-scaling-backup-service
* [3]: https://blueprints.launchpad.net/cinder/+spec/non-disruptive-backup
* [4]: https://github.com/openstack/cinder/blob/master/cinder/volume/drivers/vmware/vmdk.py#L1573
* [5]: https://review.openstack.org/#/c/193937
* [6]: https://review.openstack.org/#/c/201249
* [7]: https://review.openstack.org/#/c/237076

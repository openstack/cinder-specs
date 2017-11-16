..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Async volume migration between backends
========================================================
https://blueprints.launchpad.net/cinder/+spec/async-volume-migration-between-backends


Problem description
===================

Currently if we migrate a volume from one backend to another backend,
especially when these two backends are handling two different vendors of
arrays, we can't attach this migrating volume to server until the migration
is totally complete.

Both the dd way or driver-specific way will fail if we attach a volume when
it is migrating:

* If we migrate the volume through dd way, the source or target volume is not
  usable to server.

* If we migrate the volume through driver-specific way, the migration task is
  issued to the source backend by scheduler and the volume is owned by the
  source backend during the whole migration. If we attach this volume to a
  server when the migration is undergoing, the volume on the source
  backend(array) will be attached to the server. But after the migration is
  completed, the source volume on the source backend(array) will be deleted
  so the previous attachment will actually fails.


Use Cases
===================

Currently cinder supports four cases for volume migration:

#. Available volume migration between two different backends. These backends
   could be from one single vendor or different vendors.
#. Available volume migration between two pools of one single backend.
#. In-use(attached) volume migration using driver specific way.
#. In-use(attached) volume migration using Cinder generic migration.

This spec will only focus on case 1, to make an available volume
usable(can be attached to a server) immediately after we issue migration, no
need to wait until the migration complete. Case 2 is already well handled by
most drivers, and if we go into case 2, we won't go to case 1 again, so this
spec will focus on case 1.

Proposed change
===============

In brief, the async migration is to use some features of backend array and
I've known that these features are already supported by most vendors like
EMC VMAX, IBM SVC, HP XP, NetApp and so on. Note that for the backends not
support these features, they still can use the existing driver specific
migration or host-copy way. We won't affect anything to the existing routine,
just add a new way for developers or users to choose.

These features are:

* One array can take over other array's LUN as a remote LUN if these two
  arrays are connected with FC or iSCSI fabric.

* We can migrate a remote LUN to a local LUN and meanwhile the remote(source)
  LUN is writable and readable with the migration task is undergoing, after the
  migration is completed, the local(target) LUN is exactly the same as the
  remote(source) LUN and no data will be lost.

To enable one array to take over other array's volume, we should allow one
driver to call other drivers' interfaces directly or indirectly. These two
drivers are from two independent backends, can be managed by one single volume
node or two different volume nodes.

To enable an available volume usable(attach to a server) immediately
after we issued migration from one backend to another backend, we should
ALLOW the migration task to be send to the TARGET backend.

There will be a new interface for driver which will be called
'migrate_volume_target'. We introduce a new interface instead of use the
existing 'migrate_volume' interface because there would be lots of
differences between them, a significant difference is that the drivers
executing the 'migrate_volume' interface always take them self as the
source backend, but the new interface should take itself as the target
backend.

Some change will be made in the volume/manager.py for migrate_volume routine:

#. If not force_host_copy and new_type_id is None, firstly call source backend
   driver's migrate_volume(). If source backend driver's migrate_volume()
   return True which means it has migrated successfully, change the
   "migration_status" to "success" and update the volume, then go to step 3.
   If source backend driver's migrate_volume() returned False, go to step 2.

#. Call target backend driver's migrate_volume_target() through rpcapi, give a
   chance for target backend to perform the migration. Give target backend a
   chance to perform the migration will make the migration more flexible, and
   is important to enable async migration. migrate_volume_target() should
   return one more bool value than the migrate_volume() routine to mark the
   migration is executed synchronously or asynchronously. The whole return
   value could be: (moved, migration_async, model_update). If
   migrate_volume_target() returns moved as False which means driver can't
   perform the migration, we will go to _migrate_volume_generic() as usual to
   perform host-copy migration. If migrate_volume_target() returns moved as
   True and async as False, change the "migration_status" to "success" and
   update the volume in db, then go to step 3. If migrate_volume_target()
   returns moved as True and async as True, change the "migration_status" to
   "migrating_attachable" and update the volume in db, then just go to step 3.
   "migrating_attachable" means this volume is migrating and is safe to attach
   it to a server, and end users can check if the volume is attachable to use
   while it is migrating by "cinder show" command. Note that driver developer
   should make sure the volume on the corresponding backend is safe to
   read/write and no data corruption will occur while the migration is
   undergoing before implementing the migrate_volume_target() interface.

#. Update the volume.host to the target host and update the db with
   model_update. Now the volume is usable for server to perform read/write
   on it. If migrate_volume_target() returns async as False, the whole
   migrate_volume() routine is end now. If migrate_volume_target() returns
   True, go to step 4.

#. Call target backend driver's complete_migration_target() to monitor the
   undergoing migration, and do some cleaup after the migration is totally
   completed.

#. Update "migration_status" to "success" and end the whole migrate_volume()
   routine.


Alternatives
------------

Let users wait a long time before the volume is usable for a server until
the whole migration is totally complete.

REST API impact
---------------

None

Data model impact
-----------------

None

Security impact
---------------

None

Notifications impact
--------------------

Currently no direct notifications. But users can know that the async
migration is started when the "migration_status" changed to
"migrating_attachable" and is finished when the "migration_status"
changed to "success".

For other impact, like what operations are permitted and what operations
are not permitted, I think it's same like the existing migration.

Other end user impact
---------------------

After issued "cinder migrate" command, end users can check the
"migration_status" by "cinder show" command. If "migration_status" is
"migrating", this volume is probably not safe to attach, and if
"migration_status" is "migrating_attachable" or "success", we can attach
it safely. But one thing we should let end user know is that if
"migration_status" is "migrating_attachable", the volume is safe to
attach but the performance of the volume may not as good as other volumes
for the moment before the undergoing migration is done.

Performance Impact
------------------

As we know, driver assisted migration is mostly more efficient than host
copy, assuming that there is no read-through along with the migration.

If we attach the migrating volume and do read-through operations on the
volume, the performance of the read/write is surelly not so good as the
direct read/write. As far as I know, for the OLTP workload, the performance
may decrease no more than 15 percent.

Other deployer impact
---------------------

If deployers want to use the async feature, they should make sure the
backend driver supports this feature, and make sure these backends are
connected to each other.

Developer impact
----------------

Driver developers should implement two new interfaces: "migrate_volume_target"
and "complete_migration_target".

If drivers won't support this feature, driver devlopers needn't do anything.
We won't break down any driver or any existing function.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Wilson Liu <liuxinguo@huawei.com>

Work Items
----------

* Implement the proposed change

Dependencies
============

None

Testing
=======

* Unit-tests should be implemented

Documentation Impact
====================

None

References
==========

None

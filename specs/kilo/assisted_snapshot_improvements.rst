..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Assisted Snapshot Improvements
==========================================

https://blueprints.launchpad.net/cinder/+spec/assisted-snapshot-improvements

This spec aims to improve the infrastructure used to coordinate
between Cinder and Nova for volume snapshots.


Problem description
===================

The update_snapshot_status API is used to update fields in the Cinder
database which the driver performing a snapshot delete uses to determine
when Nova's part is finished and the Cinder driver can take over again.

This currently overloads the 'progress' field of the snapshot to
carry information from the API layer to the driver.  This is hacky and
should be reworked as more drivers are interested in supporting
assisted snapshots.

This work will also assist in supporting proper transitions between
different phases of the snapshot create/delete process as we move
toward a state machine in Kilo.

Use Cases
=========

Proposed change
===============

API<->Volume service interaction
--------------------------------

Establish a snapshot_admin_metadata table which is similar to the
volume_admin_metadata table::
    id : Integer, primary key,
    key : string,
    value : string,
    volume_id : id ref volumes.id,
    volume : relationship joining to volumes table

Setting a snapshot_status metadata value will allow the API layer
to transfer this information to the the volume service.

Volume manager<->driver interaction
-----------------------------------

Split the assisted snapshot processing code out of the driver
(currently RemoteFS-based) and into the volume manager.
This will allow drivers that support assisted snapshots to have a
call for each stage, rather than only a single delete_snapshot call.

The driver will have a property indicating it supports assisted
snapshots, which triggers use of the following methods in the manager
instead of the manager calling the driver's create_snapshot::
    create_snapshot_assisted_begin(snapshot_ref)
    create_snapshot_assisted_complete(snapshot_ref)

    delete_snapshot_assisted_begin(snapshot_ref)
    delete_snapshot_assisted_complete(snapshot_ref)

Each of these calls can return a dict of fields to be updated for the
snapshot and/or snapshot_admin_metadata by the volume manager.

This will move some database accesses currently done in the driver
to the volume manager, as well as better allowing support for defined
state transitions between the Nova and Cinder phases for tracking the
process via the new state machine work targeted for Kilo.


Alternatives
------------

Leave things as they are today, which may lead to a less robust/maintainable
infrastructure for assisted snapshots.

Data model impact
-----------------

Create new snapshot_admin_metadata table as described above.


REST API impact
---------------

update_snapshot_status() API will look for a new 'compute_snapshot_status'
field which will be used to populate the snapshot_admin_metadata entry.

If this field is not present, it will use the 'progress' field as before
for compatibility.


Security impact
---------------

None, interactions in and out of Cinder expose the same information
and level of access as today.

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

Cinder will remain backward compatible with Juno Nova for the APIs
being modified here.


Developer impact
----------------

New interfaces for drivers for creating and deleting snapshots assisted
by the compute service.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eharney

Other contributors:
  None

Work Items
----------

* Add processing to update_snapshot_status API for new fields
* Add new fields to the update_snapshot_status call made by Nova
* Add processing in snapshot-tracking code for the new compute_progress field
* Create new driver interfaces in the volume manager
* Migrate RemoteFS snapshot infrastructure to the new interfaces for the
  RemoteFSSnapDriver class.


Dependencies
============

* Have Nova send new fields for update_snapshot_status API calls
  https://review.openstack.org/#/c/134517/


Testing
=======

This will be covered by CI for GlusterFS, the NFS driver (once snapshots
are added to it in Kilo), and CI for other RemoteFS drivers.


Documentation Impact
====================

None


References
==========
* Nova change: https://review.openstack.org/#/c/134517/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
List volumes and snapshots available for manage-existing
========================================================

https://blueprints.launchpad.net/cinder/+spec/list-manage-existing

Cinder currently has the ability to take over the management of existing
volumes and snapshots ("manage existing") and to relinquish management of
volumes and snapshots ("unmanage"). The API to manage an existing volume takes
a reference, which is a driver-specific string that is used to identify the
volume on the storage backend. This process is currently confusing and error
prone. This spec's purpose is to detail APIs for listing volumes and snapshots
available for management to make this flow more user-friendly. Allowing the
listing of available volumes/snapshots will allow for the creation of an
easy-to-use GUI to migrate existing volumes to a new OpenStack environment, or
to recover in the case the Cinder database is lost.

Problem description
===================

There is no Cinder API to list the volumes and snapshots available for
managing, nor to know what reference string to use without referring to
driver-specific documentation. This means that an admin must query the storage
backend, get the appropriate references, and put them into the appropriate
Cinder API. Without the proposed APIs this flow is confusing and error-prone.

Use Cases
=========

Having Cinder manage an existing block storage volume or snapshot, using only
Cinder APIs and without referring to external driver documentation.

Proposed change
===============

Add new APIs to:
1) list volumes available for management in a given pool.
2) list snapshots available for management in a given pool.

The APIs would return information regarding the volumes/snapshots, provided by
the driver. In addition, for each resource, an indication will be provided if
it is safe to manage it. The cinder-volume manager will call the appropriate
driver to get a list of resources. For each, the driver will return a safety
indication, which includes if a volume is attached, or a Cinder ID if it
suspects the resource is already managed by Cinder. The manager will look up
any resources suspected already in use in the DB and mark them as not safe
(this avoids the driver accessing the DB).

A list of items will be returned to the user, consisting of:

- *ref* (string): a backend-specific reference to the volume that will be
  specified when the user wants to manage that volume.
- *size* (integer): the size of the volume, rounded up to Gigabytes, as an
  OpenStack user would see it if the volume is managed.
- *actual_size* (integer): the size of the volume in bytes, which may not be a
  precise Gigabyte multiple, as the volume was not created by OpenStack.
- *safe_to_manage* (boolean): True if a call to manage the volume specified by
  host_ref is expected to be safe, False if it is known that it will not be
  (for example, if the driver detects that the volume is attached, or
  cinder-volume detects that it already manages the resource).
- *reason_not_safe* (string): If "safe_to_manage" is False, specifies why.
- *driver-specific key-value pairs* (list of string:string): Any additional
  information that a driver may wish to return to help the admin identify the
  volumes/snapshots, for example a description, timestamps, etc.

For listing snapshots, return these additional fields:

- *source_volume_ref* (string): The backend-specific reference of the volume
  that is the source of the snapshot.
- *source_volume_id* (string): The Cinder volume ID of the volume that owns the
  snapshot if the volume is already managed by Cinder.

Alternatives
------------

Leave as is.

Data model impact
-----------------

None

REST API impact
---------------

Add two additional API extensions.

Security impact
---------------

Cinder currently has the ability to create objects in the storage pool, and
view or modify those objects. This allows the admin to view volumes not created
by Cinder. On the other hand, the admin can take the storage credentials that
Cinder has and perform the action on the storage backend itself, so there is no
real impact.


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

None

Developer impact
----------------

Driver maintainers can add the two new calls to bring this functionality to
their drivers.


Implementation
==============

I propose that list_manageable be added as a new GET action on the
os-volume-manage Cinder extension.  It would be separately specifiable in
Cinder's policy.json file to restrict access appropriately (default being
admin_api).

There would be new RPC calls to cinder-volume which would call two new driver
APIs (list_manageable_volumes and list_manageable_snapshots). I will implement
the LVM driver code as well, as a reference implementation.

Assignee(s)
-----------

Primary assignee:
  avishay

Work Items
----------

- Create APIs
- Create RPC calls
- Create driver interface
- Implement LVM reference
- Implement tempest test

Dependencies
============

None

Testing
=======

Standard unit tests and manual testing, as well as tempest test for these
proposed APIs, as well as manage and unmanage.

Documentation Impact
====================

The new APIs will be documented.


References
==========

None

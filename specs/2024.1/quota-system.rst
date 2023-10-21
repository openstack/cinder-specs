..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
New Quota System
================

https://blueprints.launchpad.net/cinder/+spec/count-resource-to-check-quota

Cinder quotas have been a constant pain for operators and cloud users. This
spec proposes a new quota system to resolve inconsistencies in tracking quota
usage.

Problem description
===================

Cinder's current quota system is based on reservations and commits/rollbacks.
When the API receives an operation that consumes quota, the request is
validated and Cinder checks that there's enough quota to perform the operation,
and finally creates reservations to ensure that this quota is not consumed by
other operations. Then when the operation finishes these reservations are
committed as used resources or rolled back if the operation failed.

The current usage and reservations for resources, e.g. number of volumes or
gigabytes, are tracked in the database as a counter in the ``quota_usages``
table, so if for whatever reason this counter doesn't match the real usage,
then users may become unable to create new resources or able to create
resources beyond their project limits.

There are many possible reasons why resource tracking can become out of sync in
Cinder, including bugs in the code and services dying during an operation.

As a workaround for quotas becoming out of sync, the Cinder service has code to
make reservations expire after a given time and also has the possibility of
resynching the quotas with a certain frequency.

Besides the impact on operators and users, the current quota implementation
also has an impact on developers, because the system of reserve/commit and
resource counting in the database has the downside of having to be very
thorough in order to always keep track of everything, making the quota system a
very manual and tedious process to code, which in turn results in hard to find
bugs, since we don't know at which point the counting went wrong.

Low level implementation details of the current quota system are also present
*everywhere* in Cinder, and almost every single area of the code needs to be
aware of the low level implementation details, making the code very verbose and
obscuring the high level logic.

As an example, we can look at the code of the ``accept`` method of the transfer
api in file ``cinder/transfer/api.py``, where, at the time of this writing, out
of the 106 lines of code that constitute the method, 70 are quota related!

Use Cases
=========

* A cloud administrator wants to prevent system capacities from being exhausted
  without notification so it limits quota systems based on the deployments
  capabilities.

* A cloud administrator wants to limit how many resources each department can
  consume.

* A Cinder contributor wants to add a new feature that creates or destroys
  resources, so it needs to write code to manage the quota.

Proposed change
===============

This spec proposes a new quota system where most low level quota details will
be hidden from developers working on features, simplifying feature development
and the chances of introducing new bugs.

This quota system will support 2 different drivers:

- ``StoredQuotaDriver``:  This will be similar to the old system, using
  counters in the DB, but instead of doing reservations and commits/rollbacks
  for every resource modification, it will only do reservations for the very
  few operations that really need it to keep track of the resources while the
  operation is in progress.

- ``DynamicQuotaDriver``: This driver will no longer store usage and resource
  tracking in the database table (``quota_usages``) and instead dynamically
  calculates each quota check based on the resources that exist in the
  database.

  Calculations will be counting for resources (e.g. ``snapshots``) or sum of
  values for sizes (e.g. ``gigabytes``).

  Just like the ``StoredQuotaDriver`` driver this will use reservations as
  little as possible.

The reason for having 2 drivers instead of a single one is because there are
trade-offs with each of the drivers, and the default will be the
``DynamicQuotaDriver`` driver, for reasons explained later in the
`Performance Impact`_ section.

The changes will try, as much as possible, to avoid over engineering the
solution focusing on the 2 new drivers and current cinder and not solve all
cases for potential different drivers that will probably never come.

Quota limits
------------

In the original implementation of quota classes (``quota_classes`` database
table) it was mentioned the possibility of supporting different classes besides
the existing ``default``, and being able to pass it via the context, but more
than 9 years after its implementation neither Nova nor Cinder support it, so
this new quota system and drivers will just focus on the ``default`` quota
class, which will be referred as *system wide defaults*, *global defaults*,
*global limits*, or just *defaults* since the term is easier to understand.

Limits from the ``quotas`` table will be referred as *per project limits*.

The effective quota limits that apply to a specific project after considering
the global and per project limits will be referred simply as *quota limits*.

The way quota limits are calculated will be kept as they are now:

- System wide quota limit defaults are stored in the ``quota_classes`` table
  under the records that have the ``default`` value in their ``class_name``
  column.

- Per project quota limit overrides that replace the system wide limits are
  optional and will be stored in the ``quotas`` table.

- A ``hard_limit`` value of ``-1`` indicates that there are no limits.

Resources
---------

The new quota system will not introduce or remove any of the existing quota
resources, so available resources for the quota limits will still be:
``volumes``, ``volumes_<volume-type>``, ``snapshots``,
``snapshots_<volume-type>``, ``gigabytes``, ``gigabytes_<volume-type>``,
``backups``, ``backup_gigabytes``, ``groups``, and ``per_volume_gigabytes``.
And quota usage will report in-use and reserved values for all existing limits
except the ``per_volume_gigabytes``, since there cannot be any usage for it.

Reservation values will be stored in the ``delta`` field of the
``reservations`` table just like they are today.

For the ``DynamicQuotaDriver`` these values will be dynamically added,
grouping by ``resource`` for non deleted rows belonging to the specific
project.  On the other hand the ``StoredQuotaDriver`` will track the sum in
the ``reserved`` field of the ``quota_usages`` table.

Both drivers will adhere to the following rules when reporting in-use values:

- ``volumes`` and ``volumes_<volume-type>`` quotas must match the number of non
  deleted rows in the ``volumes`` table, with the ``use_quota`` field set to
  ``true`` plus the sum of positive values from the ``reservations`` table
  where the ``resource`` matches ``volumes`` or ``volumes_<volume-type>``, in
  both cases only those values belonging to the specific project.

- ``snapshots`` and ``snapshots_<volume-type>`` quotas must match the number of
  non deleted rows in the ``snapshots`` table, with the ``use_quota`` field set
  to ``true``, and belonging to the specific project.

- ``gigabytes`` and ``gigabytes_<volume_type>`` quotas must match the sum of
  the ``size`` of the quotable volumes (as defined above) incremented in the
  sum of the ``volume_size`` values of the quotable snapshots (as defined
  above) when the ``no_snapshot_gb_quota`` configuration option is set to
  ``false`` (default value) plus the sum of positive values from the
  ``reservations`` table where the ``resource`` matches ``gigabytes`` or
  ``gigabytes_<volume-type>``, in both cases only those values belonging to the
  specific project.

- ``groups`` quota must match the number of non deleted rows in the ``groups``
  table.

- ``backups`` quota must match the number of non deleted rows in the
  ``backups`` table, and ``backup_gigabytes`` must match the sum of their
  ``size`` values.

- ``per_volume_gigabytes`` is a quota limit that doesn't need any kind of
  calculation.

Mechanism
---------

The new quota system will rely heavily on database transactions and database
row locking using the ``SELECT FOR UPDATE`` SQL statement to control parallel
operations and ensure quota limits are honored and **all** database changes
happen or they are automatically rolled back.

A high level view of how this mechanism would work is:

- Start a transaction
- Get current quota limits creating a lock on those rows
- Check operation doesn't go over quota
- Create the resource on the database or make reservations
- Finish the transaction releasing the lock

The lock will only happen on the rows of the resources we are interested in,
allowing operations on other projects and resources to be executed in parallel.
For example, quota checks to create a volume will lock rows for ``volumes``,
``volumes_<volume-type>``, ``gigabytes``, and ``gigabytes_<volume-type>``, so
cinder will be able to check for the quota to create a backup since that only
requires ``backup`` and ``backup_gigabytes``.

The new system will leverage Python context manager functionality and the Oslo
DB transaction context provider available in the Cinder ``RequestContext``
(``context.session``) to facilitate the sharing of the transaction/session
between different areas of the Cinder code.

This will allow developers to write cleaner code, for example when creating a
volume, the ``create`` method of the Cinder ``Volume`` Oslo object will have to
check that it can create 1 volume, that will consume additional gigabytes and
that the size of the volume doesn't exceed the largest volume size allowed, so
the code will be something like this:

.. code-block:: python

   with self.quota_check(self._context, self.volume_type.id,
                         vol_gbs=self.size,
                         vol_qty=1,
                         vol_size=self.size):

      db_volume = db.volume_create(self._context, updates)

The ``quota_check`` is a property in the ``Volume`` OVO that returns a context
manager that ensures quota limits are honored.  Returned context manager
depends on whether the volume consumes quota or not, returning a *noop* if it
doesn't and returning a context manager provided by the quota driver if it
does.

The quota driver context manager starts a DB session/transaction in the
provided ``context`` so the ``volume_create`` call will use that same session
to create the volume record, and the transaction will be finalized when the
code exits the context manager, thus ensuring that no other operations check
the quota until the volume has been created.

From a developer's point of view all this will be hidden, because at a higher
level all they need to do is create the ``Volume`` OVO and the quota will be
automatically checked.  As an example this is the code in the create volume
flow (*cinder/volume/flows/api/create_volume.py*):

.. code-block:: python

   volume = objects.Volume(context=context, **volume_properties)
   volume.create()

In an effort to abstract the quota system implementation and hide its details
from most of the code, the code interfacing with the driver directly will no
longer use the resource names such as ``gigabytes`` and
``volumes_<volume-type>``, instead the parameters that will be used for the
volume and snapshot context manager checker are:

- ``vol_qty``: Delta on the number of volumes that will be consumed within the
  checker context manager.  The quota system internal name for this is
  ``volumes`` in the database.

- ``vol_type_vol_qty``: Delta on the number of volumes for the specific volume
  type that will be consumed within the checker context manager.  Defaults to
  the value of ``vol_qty`` since that's the most common case. The quota system
  internal name for this is ``volumes_<volume-type>`` in the database.

- ``vol_gbs``: Delta on the number of volume gigabytes that will be consumed
  within the checker context manager.  The quota system internal name for this
  is ``gigabytes`` in the database.

- ``vol_type_gbs``: Delta on the number of volume gigabytes for the specific
  volume type that will be consumed within the checker context manager.
  Defaults to ``vol_gbs`` since that's the most common case. The quota system
  internal name for this is ``gigabytes_<volume-type>`` in the database.

- ``snap_qty``: Delta on the number of snapshots that will be consumed within
  the checker context manager.  The quota system internal name for this is
  ``snapshots`` in the database.

- ``snap_type_qty``: Delta on the number of snapshots for the specific volume
  type that will be consumed within the checker context manager.  Defaults to
  the value of ``snap_qty`` since that's the most common case. The quota
  system internal name for this is ``snapshots_<volume-type>`` in the database.

- ``snap_gbs``: Delta on the number of snapshot gigabytes that will be consumed
  within the checker context manager.  Will end up using the quota system
  internal name of ``gigabytes`` if the ``no_snapshot_gb_quota`` configuration
  option is set to ``false`` (default) or will be disregarded if set to
  ``true``.

- ``snap_type_gbs``: Delta on the number of snapshot gigabytes for the specific
  volume type that will be consumed within the checker context manager.
  Defaults to the value of ``snap_gbs`` since that's the most common case.
  The quota system internal name for this is ``gigabytes_<volume_type>`` in the
  database if the ``no_snapshot_gb_quota`` configuration option is set to
  ``false`` or will be disregarded if set to ``true``.

- ``vol_size``: Total volume size when creating or extending it, in the
  internally the quota system uses the ``per_volume_gigabytes`` quota limit to
  check this value.

This change may seem worthless, but it has its value, because it abstracts the
implementation details of the snapshots and volumes sharing the same quota size
limits which provides:

- Cleaner code since snapshot creation or transfer of a volume with snapshots
  doesn't need to know about the ``no_snapshot_gb_quota`` configuration option.

- If we want to add, in the future, snapshot specific quota limits -
  ``snapshot_gigabytes`` and ``snapshot_gigabytes_<volume-type>``- we'll be
  able to do so without affecting any of the Cinder code with the sole
  exception of the quota driver itself.

Reservations
------------

For the new quota system the reservation commit and rollback operations will be
grouped into a single context manager that handles both cases.  Committing and
rolling back reservations have different meanings for the 2 drivers.

For the ``DynamicQuotaDriver`` these are *noop* operations, since checks use
the DB values every time and the database has already been modified in the
same transaction that the reservations are removed.  On the other hand the
``StoredQuotaDriver`` needs to modify the ``in_use`` and ``reserved`` counters
in the ``quota_usages`` table accordingly to the operation.

As mentioned before, reservations will only be necessary for specific
operations, to be exact on 3 operations: extend, transfer, and retype.

Each of these operations have different reasons for requiring reservations:

- Extend: Until the operations completes, the ``size`` field of the volume in
  the database must be kept as it is to reflect its real value, but we need to
  reserve the additional gigabytes, for ``gigabytes`` and
  ``gigabytes_<volume_type>`` quotas, during the operation so we don't go over
  quota due to other concurrent operations.  If the operation completes
  successfully the ``size`` of the volume will be increased and the
  reservations will be committed.

- Transfer: Under normal circumstances accepting a transfer would not require
  the use of a reservation, as we should be able to check the quota and do the
  database changes to accept the transfer in the same transaction.
  Unfortunately the *SolidFire* and *VMDK* drivers need to make some changes in
  their backend on transfer, so the volume service has to make a driver call.

  We cannot keep the database locked while the driver call completes, as it can
  take some time and we don't want to prevent the API from processing other
  operations.

  That is why reservations will be created before calling the driver and
  cleared after accepting the resources.

  In terms of reservations, transfers are complex for the
  ``StoredQuotaDriver``, because when completing one it needs to modify 2
  different projects.  One to increase counters and the other to decrease them,
  so higher levels will need to make 2 different calls for 2 different
  projects, one with positive and one with negative numbers and negative
  numbers should ignore quota usage and limits.

  When storing reservations for transfer of volumes with snapshots they have
  to be stored separately in case someone restarts the service after changing
  the ``no_snapshot_gb_quota_toggled`` configuration option as detailed in the
  ``vol_snap_check_and_reserve_cm`` method below.

- Retype: When doing a retype the API needs to reserve
  ``gigabytes_<dest-volume-type>`` and ``volumes_<dest-volume-type>`` until the
  operation is completed as well as create negative reservations for
  ``gigabytes_<source-volume-type`` and ``volumes_<source-volume-type>``.

  This consumes volumes and gigabytes on both types until the operation
  completes for the following reasons:

  * If the retype fails we will continue consuming volumes and gigabytes on the
    source volume type, but if we "released" that usage when we started the
    operation we may find that there is no longer enough quota available for
    the volume to stay there.  This is the main reason.

  * Even if the retype succeeds Cinder doesn't know the reasons why the cloud
    administrator has set the quota limits, so freeing the source gigabytes and
    volumes as soon as the retype starts means that if a new volume for the
    source type is created during the retype Cinder will be exceeding the quota
    for that volume type.

  This is the only operation where a race condition can happen, though it's a
  corner case.  It can happen if we are adding a new quota limit, global or per
  project, to a volume type resource (e.g. ``volume_<volume_type>``) that
  didn't have any limit in the database while at the same time we are doing
  volume retypes to that same volume type.  This race should fall within
  reasonable expectations, as one would argue that the limit was added right
  after the retype already passed the quota check.

It is possible that while doing an operation on a resource the code flow
doesn't complete in an unexpected way leaving leftover reservations in the
database, for example:

- A coding bug in Cinder that leaves the volume in an unexpected status.
- Service kill.
- Node restart or shutdown.

For these situations the new quota system will add code to the
``os-reset_status`` REST API action on volumes to automatically clear any
reservations that the volume may have when the status is changed, which is what
happens when a volume is stuck in ``extending``, ``retyping``, etc.  This way
there is no need to wait until the reservation expires and the operator can do
the cleanup in an easy way without needing additional API calls.

On volume deletion the code will also clear any existing reservations on the
specific volume.

To facilitate the cleanup of these reservations the volume's id will be used as
the ``uuid`` field for all the reservations, instead of creating a random one,
regardless of the value of the ``resource`` field in the ``reservations``
table.

Both drivers will create reservations the same way to facilitate switching the
drivers without having usage numbers go out of sync.

Changing configuration
----------------------

There are 2 Cinder configuration options that are crucial for the new quota
system to operate correctly: ``quota_driver`` and ``no_snapshot_gb_quota``.

The ``no_snapshot_gb_quota`` configuration option is used to determine whether
snapshots should be counted towards the volume quota or not, so this is not
something we want to be counting in some places and not counting in others; we
want a consistent behaviour through **all** the Cinder services, which means
that they must have the same value.

Currently Cinder has no way of enforcing the same value for the
``no_snapshot_gb_quota``, and what's worse, it cannot even know when the
current quota calculations have become invalid because this configuration
option has changed (`Bug #1952635
<https://bugs.launchpad.net/cinder/+bug/1952635>`_).

This is something we definitely don't want in the quota system, and with the
new quota system we have bigger problems, because it's not only
``no_snapshot_gb_quota`` that can be changed, but also ``quota_driver``, and
changing the quota driver means that a quota system may need to recalculate
things to ensure that it starts operating with the correct quota assumptions.
For example when changing from the ``DynamicQuotaDriver`` to the
``StoredQuotaDriver`` all the counters in the DB will be wrong, so the
``StoredQuotaDriver`` needs to calculate the counters before it can start
working or the whole quota system will not operate correctly.

These configuration options are not the kind of things that are frequently
changed, and we expect most deployments to never have to change them at all,
but Cinder should still provide a way for them to be safely changed since one
of the cases we expect to happen is a deployment outgrowing the usefulness of
the ``DynamicQuotaDriver`` and running into performance issues.  In that case
they will want to switch to the ``StoredQuotaDriver``.

To support changing configuration option changes to the quota system there are
3 things that the new quota system needs to be able to do:

- Detect changes in configuration options.

- Signal drivers that the ``no_snapshot_gb_quota`` configuration option has
  changed and for driver to react to this change.

- Signal drivers that they were not the quota driver that was running on the
  last start and they should see if they need to do some calculations.

To detect changes to these configuration options, a new ``global_data`` table
will be created to store the currently used configuration values.  This table
will be used to signal quota drivers when things have changed.

A system administrator will have to follow these steps to change any of these 2
configuration options:

- Stop all Cinder services.

- Change the cinder.conf file in all the nodes where a Cinder service is
  running.

- Run a cinder-manage command to apply the changed options.

- Restart Cinder services.

The cinder-manage command will not only trigger the quota system
recalculations, but it will also make the appropriate DB changes in the
``global_data`` table to reflect the new configuration options that are in
effect.

Since we cannot allow Cinder services to run with mismatching configuration
options they will fail to start if the quota configuration options from the
database don't match the one that the service has.  This will prevent system
administrators making an error and only realize it after their whole system has
some crazy quotas.

Please see the `Changing configuration alternatives`_ for other possible
mechanism to the one proposed here.

Interface
---------

Here is the proposed interface for the new quota system drivers:

NAME
****

Unique string of maximum 254 ASCII characters that identifies the driver.

__init__
********

.. code-block:: python

   def __init__(self, driver_switched, no_snapshot_gb_quota_toggled):

Initialization method for the quota driver where the ``driver_switched``
parameter indicates whether the last run was done using the same Quota driver
or if a different one was used and this is the first run with this one.

This is important because switching to the ``StoredQuotaDriver`` from the
``DynamicQuotaDriver`` means that ``in-use`` and ``reserved`` counters need to
be recalculated since they could be out of sync or missing altogether.

This effort is going to focus on only supporting these 2 quota drivers and
avoid unnecessary complexity, because if we wanted to support other kind of
drivers that were not based on the Cinder database we would need to add a more
complex mechanism, since the cinder code would need to notify drivers when the
limits are changed in the DB and there would need to be a way for Cinder to
request information from the old quota driver, such as current reservations,
when switching.

The interface can be enhanced if a future quota driver finds it insufficient.

The ``no_snapshot_gb_quota_toggled`` parameter indicates whether the option has
changed since the last run.  This is important for the ``StoredQuotaDriver``
that would need to recalculate ``in-use`` and ``reserved`` counters.  This is
something that doesn't work correctly right now.

Drivers can block the Cinder database when synchronizing when the driver has
been switched or the snapshot quota configuration option has been toggled,
because the driver will only be called with any of the parameters set to
``True`` on a single service in the deployment and the quota will not be in use
at that time by any other service.

resync
******

.. code-block:: python

    def resync(self, context, project_id):

This is only relevant for the ``StoredQuotaDriver``, and is intended to allow
the ``cinder-manage`` command request a recalculation of quotas for a specific
project or for the whole deployment.

set_defaults
************

.. code-block:: python

    def set_defaults(self, context, **defaults):

Set system wide default limits.

The keys for the keyword arguments ``defaults`` are in the internal form of the
quota system, that is to say, they will be ``gigabytes`` and not
``vol_gbs``.

This will be a common implementation for both database based quota drivers,
where it modifies the record if it exists and creates it if it doesn't.

set_limits
**********

.. code-block:: python

    def set_limits(self, context, project_id=None, **limits):

Set project specific limits.

The keys for the keyword arguments ``limits`` are in the internal form of the
quota system, that is to say, they will be ``gigabytes`` and not ``vol_gbs``.

This will be a common implementation for both database based quota drivers,
where it modifies the record if it exists and creates it if it doesn't.

clear_limits_and_defaults_cm
****************************

.. code-block:: python

    def clear_limits_and_defaults_cm(self, context,
                                     project_id=None, type_name=None):

This context manager removes, on exit, all existing per project limits, for
when a project is deleted, or all type specific global defaults and per project
limits, for when a type is deleted.

Parameters ``project_id`` and ``type_name`` will be used used as filter in the
deletion.  So if only ``project_id`` is provided then only per project entries
will be deleted (in the db driver those from the ``quotas`` table), and if
only the ``type_name`` is provided then only ``gigabytes_<type-name>``,
``volume_<type-name>`` and ``snapshots_<type-name>`` resources will be removed
but for per-project (``quotas`` db table) and global (``quota_classes`` table).

If an error occurs within the context manager the limits and defaults should
not be cleared.

This will be a common implementation for both database based quota drivers.

type_name_change_cm
*******************

.. code-block:: python

    def type_name_change_cm(self, context, old_name, new_name,
                            project_id=None):

Context manager to make necessary modification, on enter, to system wide
defaults and per project limits to account for a volume type name change.

This will rename ``gigabytes_<old_name>``, ``volume_<old_name>`` and
``snapshots_<old_name>`` to ``gigabytes_<new_name>``, ``volume_<new_name>`` and
``snapshots_<new_name>`` respectively in all tables.

The database change to the volume type name is called within this context
manager to ensure that the quota defaults and limits stay in sync with the
volume type name and we don't change one but not the other.

This will be a common implementation for both database based quota drivers.

get_defaults
************

.. code-block:: python

   def get_defaults(self, context, project_id=None):

Returns system wide defaults for quota limits. If ``project_id`` is not
``None`` then volume types quota resources (``volumes_<volume-type>``,
``gigabytes_<volume-type>``, and ``snapshots_<volume-type>``) will be filtered
based on the project's visibility of the volume types, if ``project_id`` is
``None`` then **all** defaults will be returned regardless of the ``is_public``
value of the volume types.

In terms of volume type visibility, a project can view all public volume types
and private ones where it has permissions (entries in the
``volume_type_projects`` table).

System wide defaults are stored in the database in the ``quota_classes`` table
with the ``default`` value on the ``class_name``.

Returned data is a dictionary mapping resources to their hard limits, and must
include **all** volume type resources even if there is no record in the
database.

In the following example of returned data the ``gigabytes_lvmdriver-1``,
``volumes_lvmdriver-1``, and ``snapshots_lvmdriver-1`` are not present in the
database:

.. code-block:: python

   {
    "per_volume_gigabytes": -1,
    "volumes": 10,
    "gigabytes": 1000,
    "snapshots": 10,
    "backups": 10,
    "backup_gigabytes": 1000,
    "groups": 10,
    "gigabytes___DEFAULT__": -1,
    "volumes___DEFAULT__": -1,
    "snapshots___DEFAULT__": -1,
    "gigabytes_lvmdriver-1": -1,
    "volumes_lvmdriver-1": -1,
    "snapshots_lvmdriver-1": -1
   }

This will be a common implementation for both database based quota drivers.

get_limits_and_usage
********************

.. code-block:: python

   def get_limits_and_usage(self, context, project_id, usages=True):

Get **all** effective quota limits for a specific project, and optionally quota
usage, for a specific project.  If ``project_id`` is ``None`` the one from the
``context`` will be used.

Volume types quota resources (``volumes_<volume-type>``,
``gigabytes_<volume-type>``, and ``snapshots_<volume-type>``) will be filtered
based on the project's visibility of the volume types.

A project can view all public volume types and private volume types where it
has permissions (entries in the ``volume_type_projects`` table).

A quota limit values defined in the ``quotas`` table overrides global values
from the ``quota_classes``.

Returned data will always be a dictionary (or ``defaultdict``), but the
contents will depend on whether we are getting quota usage or not.  Just like
the ``get_defaults`` method this returns **all** volume type resources even if
there is no record in the database.

.. code-block:: python

   {
    "per_volume_gigabytes": -1,
    "volumes": 8,
    "gigabytes": 1000,
    "snapshots": 10,
    "backups": 10,
    "backup_gigabytes": 1000,
    "groups": 10,
    "gigabytes___DEFAULT__": -1,
    "volumes___DEFAULT__": -1,
    "snapshots___DEFAULT__": -1,
    "gigabytes_lvmdriver-1": -1,
    "volumes_lvmdriver-1": -1,
    "snapshots_lvmdriver-1": -1
   }

With quota usage returned value will look like this:

.. code-block:: python

    {
     'per_volume_gigabytes': {'limit': -1, 'in_use': 0, 'reserved': 0},
     'volumes': {'limit': 8, 'in_use': 1, 'reserved': 0},
     'gigabytes': {'limit': 1000, 'in_use': 1, 'reserved': 0},
     'snapshots': {'limit': 10, 'in_use': 0, 'reserved': 0},
     'backups': {'limit': 10, 'in_use': 0, 'reserved': 0},
     'backup_gigabytes': {'limit': 1000, 'in_use': 0, 'reserved': 0},
     'groups': {'limit': 10, 'in_use': 0, 'reserved': 0},
     'gigabytes___DEFAULT__': {'limit': -1, 'in_use': 0, 'reserved': 0},
     'volumes___DEFAULT__': {'limit': -1, 'in_use': 0, 'reserved': 0},
     'snapshots___DEFAULT__': {'limit': -1, 'in_use': 0, 'reserved': 0},
     'gigabytes_lvmdriver-1': {'limit': -1, 'in_use': 1, 'reserved': 0},
     'volumes_lvmdriver-1': {'limit': -1, 'in_use': 1, 'reserved': 0},
     'snapshots_lvmdriver-1': {'limit': -1, 'in_use': 0, 'reserved': 0}
    }

group_check_cm
**************

.. code-block:: python

   def group_check_cm(self, context, qty=1, project_id=None):

Context manager to check group quota upon context entering.

Raises ``QuotaError`` if quota usage would go over the quota limits upon adding
``qty`` new groups.

Effective quota limits are determined based on the project's quota limits
(``hard_limit`` for the ``groups`` resource in the ``quotas`` table) if defined
or the global defaults (in the ``quota_classes`` table) otherwise.

The project is determined by the ``project_id`` parameter or the ``context``'s
``project_id`` if the optional ``project_id`` parameter value is ``None``.

The context manager must ensure that there are no race conditions with
concurrent calls to ``group_check_cm`` within different threads and processes
in the node as well as across different nodes.

For the database driver this can be achieved using a ``SELECT FOR UPDATE`` on
the ``groups`` quota limit which blocks other requests until the context
manager exists.

Users of this context manager should try to keep the code within the context
manager to a minimum to allow higher concurrency.

For the DB driver, the context manager will start a database
transaction/session, making it available in the ``session`` attribute of the
provided ``context``, and this transaction will be committed if the code
enveloped by the context manager completes successfully, but if an exception is
raised in the enveloped code then the transaction will be rolled back.  So this
context manager not only checks the quotas but also provides a transaction
context.

An example of using this context manager within the ``create`` method of the
``Group`` Oslo Versioned Object:

.. code-block:: python

   with quota.driver.group_check_cm(self._context, qty=1):

       db_groups = db.group_create(self._context,
                                   updates,
                                   group_snapshot_id,
                                   source_group_id)

group_free
**********

.. code-block:: python

   def group_free(self, context, gbs, qty=1, project_id=None):

Context manager to free group quotas upon context exiting.  The DB row soft
deletion of groups will be enclosed by this call.

This is only relevant for the ``StoredQuotaDriver`` that needs to decrease its
counters.

backup_check_cm
***************

.. code-block:: python

   def backup_check_cm(self, context, gbs, qty=1, project_id=None):

Context manager to check backup quotas upon context entering.

Raises ``QuotaError`` if quota usage would go over the quota limits upon adding
``qty`` backups or ``gbs`` backup gigabytes.

Effective quota limits are determined based on the project's quota limits
(``hard_limit`` for the ``backups`` and ``backup_gigabytes`` resources in the
``quotas`` table) if defined or the global defaults (in the ``quota_classes``
table) otherwise.

The project is determined by the ``project_id`` parameter or the ``context``'s
``project_id`` if the optional ``project_id`` parameter value is ``None``.

The context manager must ensure that there are no race conditions with
concurrent calls to ``backup_check_cm`` within different threads and processes
in the node as well as across different nodes.

For the database driver this can be achieved using a ``SELECT FOR UPDATE`` on
the ``backups`` and ``backup_gigabytes`` quota limits which blocks other
requests until the context manager exists.

Users of this context manager should try to keep the code within the context
manager to a minimum to allow higher concurrency.

For the DB driver, the context manager will start a database
transaction/session, making it available in the ``session`` attribute of the
provided ``context``, and this transaction will be committed if the code
enveloped by the context manager completes successfully, but if an exception is
raised in the enveloped code then the transaction will be rolled back.  So this
context manager not only checks the quotas but also provides a transaction
context.

An example of using this context manager within the ``create`` method of the
``Backup`` Oslo Versioned Object:

.. code-block:: python

   with quota.driver.backup_check_cm(self._context, qty=1, gbs=self.size):

       db_backup = db.backup_create(self._context, updates)

backup_free
***********

.. code-block:: python

   def backup_free(self, context, gbs, qty=1, project_id=None):

Context manager to free backup quotas upon context exiting.  The DB row soft
deletion of the backup will be enclosed by this call.

This is only relevant for the ``StoredQuotaDriver`` that needs to decrease its
counters.

vol_snap_check_and_reserve_cm
*****************************

.. code-block:: python

   def vol_snap_check_and_reserve_cm(self, context, type_id, type_name=None,
                                     project_id=None,
                                     *,
                                     uuid=None,
                                     vol_gbs=0, vol_qty=0,
                                     vol_type_gbs=None, vol_type_vol_qty=None,
                                     snap_gbs=0, snap_qty=0,
                                     snap_type_gbs=None, snap_type_qty=None,
                                     vol_size=0):

Context manager that, upon entering, checks volume and snapshot quotas and
optionally makes reservations.

Volumes and snapshot are tightly coupled resources, since a snapshot cannot
exist without a parent volume, so their quota checks are handled jointly in the
same method.

Raises ``QuotaError`` if quota usage would go over the quota limits upon
consuming provided resources:

- ``vol_qty`` number of volumes being reserved.

- ``vol_gbs`` additional volume gigabytes.

- ``vol_type_vol_qty`` volumes of the specified volume type. Defaults to the
  value of ``vol_qty``.

- ``vol_type_gbs`` additional volume gigabytes of the specified volume type.
  Defaults to the value of ``vol_gbs``.

- ``snap_qty`` snapshots.

- ``snap_gbs`` additional snapshot gigabytes.

- ``snap_type_qty`` snapshots of the specified volume type. Defaults to the
  value of ``snap_qty``.

- ``snap_type_gbs`` additional snapshot gigabytes of the specified volume type.
  Defaults to the value of ``snap_gbs``.

Unlike the ``vol_gbs``, ``vol_type_gbs``, ``snap_gbs``, and ``snap_type_gbs``
parameters, the ``vol_size`` is not an increment over existing consumption, but
an absolute value representing the total size of the volume.  And the context
manager also raises a ``QuotaError`` exception if it is greater than the
``per_volume_gigabytes`` limit.

Effective quota limits are determined based on the project's quota limits
``volumes``, ``volumes_<volume-type>``, ``snapshots``,
``snapshots_<volume-type>``, ``gigabytes``, ``gigabytes_<volume_type>``, and
``per_volume_gigabytes`` if defined in the ``quotas`` table or the global
defaults defined in the ``quota_classes`` table otherwise.

The project is determined by the ``project_id`` parameter or the ``context``'s
``project_id`` if the optional ``project_id`` parameter value is ``None``.

The volume type name (``type_name``) is necessary to perform quota checks, but
the method can query this information based on the ``type_id``.  Due to current
Cinder behavior (where a type can be changed to private even when projects have
volumes) then the quota driver needs to confirm that the project still has
access to it.

Volumes and snapshots are currently the only resources that can have
reservations, and this method automatically creates them when a ``uuid`` is
provided.  This ``uuid`` must be of the primary resource for the operation,
that is to say that if we are transferring a volume with all its snapshots the
reservations will pass the volume's ``uuid``.

Both drivers must use different entries for volume and snapshot gigabyte
reservations because the ``no_snapshot_gb_quota_toggled`` configuration option
may be changed and the service restarted before a transfer is accepted, and the
``StoredQuotaDriver`` will need to make a decision both when recalculating (if
driver has changed) and on transfer accept.

This context manager must ensure that there are no race conditions with
concurrent calls to ``vol_snap_check_and_reserve_cm`` within different threads
and processes in the node as well as across different nodes.

For the database driver this can be achieved using a ``SELECT FOR UPDATE`` on
the ``volumes``, ``volumes_<volume-type>``, ``snapshots``,
``snapshots_<volume-type>``, ``gigabytes`` and ``gigabytes_<volume_type>``
quota limits which blocks other volume and snapshot requests until the context
manager exists.

Users of this context manager should try to keep the code within the context
manager to a minimum to allow higher concurrency.

When creating reservations the context manager must ensure that they are
cleaned up if an exception is raised within the context manager. For the DB
driver the context manager will start a database transaction/session, making
it available in provided ``context``, and will commit everything on normal
context manager exit and roll everything back, including the reservations, when
an exception is raised.

An example of using this context manager within the ``create`` method of the
``Volume`` Oslo Versioned Object:

.. code-block:: python

   with self.quota_check(self._context, self.volume_type.id,
                         volume_type and volume_type.name,
                         vol_gbs=self.size,
                         vol_qty=1,
                         vol_size=self.size):

       db_volume = db.volume_create(self._context, updates)

Where ``quota_check`` is a property that takes into account whether the volume
uses quota or not:

.. code-block:: python

   @property
   def quota_check(self):
       if self.get('use_quota', True):
           return quota.driver.vol_snap_check_and_reserve
       return self.nullcontext

vol_snap_free
*************

.. code-block:: python

   def vol_snap_free(self, context, type_id, type_name=None, project_id=None,
                     *,
                     vol_gbs=0, vol_qty=0,
                     vol_type_gbs=None, vol_type_vol_qty=None,
                     snap_gbs=0, snap_qty=0,
                     snap_type_gbs=None, snap_type_qty=None):

Context manager to free volume and snapshot quotas upon context exiting.

This is only relevant for the ``StoredQuotaDriver`` that needs to decrease its
counters.

reservations_clean_cm
*********************

.. code-block:: python

   def reservations_clean_cm(self, context, resource_uuid, commit=True):

Context manager that cleans all reservations, committing or rolling back, for
the given uuid on **exit**.

The ``uuid`` is the "primary" uuid of the operation and it won't be a different
uuid for each resource that has been reserved. E.g. when accepting a volume
transfer with its snapshots, all reservations will use the volume's id.

For the ``DynamicQuotaDriver`` this is mostly just deleting the entries from
the database, but for the ``StoredQuotaDriver`` it needs to adjust the
``in-use`` and ``reserved`` counters.

These counters may be from different projects, for the transfer of volumes, so
the ``context``'s ``project_id`` will be ignored.

The ``DynamicQuotaDriver`` driver must also take into account the
``no_snapshot_gb_quota_toggled`` configuration option when committing a
transfer, because the snapshot reservations are stored in different row entries
in case the option is changed and the service rebooted before a transfer is
accepted.

Differences
-----------

There are some differences between the new and old system that are worth
highlighting:

- Resource consumption rules stated in the *Resources* section of this spec are
  absolute, so it doesn't matter if a volume becomes in ``error`` status
  because scheduling failed or because the driver call on the volume service
  failed. If there is a quotable database record, then it will be counted
  towards the quota.

- Negative reservations, created when retyping a volume, won't be taken into
  account in usage calculations, because like we mentioned before we want the
  ``volumes_<volume-type>`` and ``gigabytes_<volume-type>`` of the source type
  to still be consumed while we do the operations, since we don't know if we'll
  succeed or not, and on failure we would need to consume them again.

- The new quota system drops the illusion that Cinder can support multiple ORM
  systems and accepts the fact that Cinder is tightly coupled with SQLAlchemy
  and MySQL/InnoDB (this is not new, `there is already a patch proposed that
  removes the intermediate layer
  <https://review.opendev.org/c/openstack/cinder/+/813229>`_, so instead of
  having all the quota code in ``cinder/db/sqlalchemy/api.py`` it will be under
  ``cinder/quota``, including all the database queries.

  This approach has the downside of having DB code in multiple places, with
  potential code duplication, but on the other hand it has the great benefit of
  having the quota code contained in fewer files and using less memory for
  custom quota drivers (currently the standard quota driver file is always
  loaded even if it's not instantiated).

- All deployments will use the default quota class instead of supporting the
  already deprecated configuration file quota limits.

- The new quota system fixes a number of existing bugs, so there are some
  undesired behaviors that change:

  * Now listing quota limits and quota usages won't show private types the
    project doesn't have access to (`bug #1576717
    <https://bugs.launchpad.net/cinder/+bug/1576717>`_).

  * A limit of 0 will be shown if a project has resources for a type we no
    longer have access to because it was made private after the resource was
    created (`related bug #1952456
    <https://bugs.launchpad.net/cinder/+bug/1952456>`_).  This can also happen
    if an admin creates a volume for a private type that the project doesn't
    have access to.

Limitations
-----------

* Spec is aimed at these 2 drivers, so additional drivers may not be easy to
  add.  Though it shouldn't be a problem if these 2 work as expected.

* There is a bottleneck in concurrent code execution, because the code locks on
  the system wide defaults, which are common to **all** projects. So, even if
  the critical section code enveloped by the check context managers is very
  small, it will still limit to only 1 context entering at a time for the whole
  deployment for the given quota limits.  As an example, if we are concurrently
  creating 100 volumes in as many projects, they will be happening mostly in
  parallel, but once they reach the point to check quota limits and create the
  DB record they will be serialized.

* Race condition on the retype operation as explained in the *reservations*
  section.

Alternatives
------------

Work with what we have
**********************

Some of the alternatives include:

- Carefully go through the Cinder code looking for potential error causes and
  fixing them.

- Refactor existing quota code to move part of the Quota Python logic into
  database queries.

- Refactor existing code to reduce spillage of the quota system implementation
  details all over the code and reduce the usage of reservations to only the
  strictly necessary cases.

These alternatives have the same underlying issue as the current
implementation, where it would be hard to tell whether we have resolved all the
issues or not, and upon encountering another out of sync case in a deployment
we would be, once again, in a position where we cannot tell how we reached that
point.

Unified Limits
**************

Another alternative is to use the `KeyStone Unified Limits
<https://docs.openstack.org/keystone/latest/admin/unified-limits.html>`_.  At
first glance this may seem like a perfect solution since it:

* Allows for a unified limit system in all OpenStack (once all projects
  implement it).

* Supports different enforcement models, including hierarchies.

But upon closer inspection it's not without its disadvantages:

* While `Glance
  <https://review.opendev.org/q/topic:bp%252Fglance-unified-quotas>`_ and `Nova
  <https://review.opendev.org/q/topic:bp%252Funified-limits-nova>`_ implemented
  its usage in the Yoga release this still cannot be considered a *proven
  solution* since there have not been enough time for users to actually
  evaluate it.

* The Unified Limits system does not have any mechanism to prevent race
  conditions between concurrent operations.  So we'll have to implement our own
  mechanism that needs to work across all the Cinder services. It can be with a
  DLM, some database locking, or how Nova is going to do it, which is to check
  the limit, commit claim, then check limit again and revert if over usage is
  detected.  The Nova mechanism means that we are always doing a double check
  and sometimes a revert, and we can even get a false failure due to a double
  race condition on the checks (2 concurrent requests pass the initial check
  and then both fail on the confirmation check, whereas only 1 of them on its
  own would have succeeded).

* The oslo.limit project will fail a limit check if the limit has not been
  previously registered in KeyStone, which is the opposite of how our quota
  system currently behaves, as it assumes unlimited (-1).  This means that
  Cinder will either have to manage the registrations of the limits when we
  create or destroy volume types, when a project is given access to a volume
  type, when a volume type's public status changes, etc. or to force operators
  to manage all this on their own. A more reasonable alternative would be to
  modify the oslo.limit project to support alternative behavior on non defined
  values.

* It will be slower since we have to call an external service, KeyStone, for
  the limit check, which has to check the user making the call, go to the
  database, etc.  And because each of the resources that are checked `requires
  its own REST API call to KeyStone
  <https://github.com/openstack/oslo.limit/blob/a6fff3be3194ebb26c5c851ddb0200f84458c46d/oslo_limit/limit.py#L288>`_

  This could be improved in Keystone to allow multiple simultaneous checks.

* Hierarchical support using the Strict Two Level enforce mechanism `isn't
  implemented in oslo.limit
  <https://github.com/openstack/oslo.limit/blob/a6fff3be3194ebb26c5c851ddb0200f84458c46d/oslo_limit/limit.py#L201-L215>`_

Fix bottleneck
**************

As mentioned before, in the proposed quota system there is a bottleneck in the
concurrent code execution due to the DB locking, because it is locking using
entries from the ``quotas`` table which are shared among all projects.

To resolve this bottleneck using the DB locking we would need to duplicate the
system wide defaults.  These entries can be duplicated in the ``quotas`` table
or in the ``quota_classes`` table.

If they are duplicated into the ``quotas`` table, then a new column would need
to be added (``is_default``) to flag the contents as being defaults or not.
Because when a global default from the ``quota_classes`` table is changed it
would need to be changed in the ``quotas`` table records that have the default
values but not on the records that have been explicitly set, even if they have
the same value as the default.

If the ``quota_classes`` table is used, then we would store the ``project_id``
into the ``name`` column, which means that we would have trouble in the future
if we ever wanted to fully implement the Quota Classes concept.  Though this is
unlikely given how long it has been since the table was created and how the
concept has never been implemented.

When setting a global default quota limit we would need to remove the
restriction on the ``name`` column being ``default`` when using the
``quota_classes`` and if we used the ``quotas`` table we would need an
additional query besides the one to the ``default`` ``quota_classes`` record,
as we would need to update non deleted records from the ``quotas`` table for
that resource that have the ``is_default`` value set to ``true``.

Another thing to consider is that Cinder doesn't know beforehand what projects
exist, so it will also need to dynamically duplicate the global default quota
records if they are not present when the first operation on a project is
called.  This can be done efficiently, only incurring into additional queries
on the first request for a project, by assuming the values exist and querying
with locking on them, and only if the result is missing the values we go and
duplicate the global defaults.

This dynamic duplication is also tricky, because we don't want to have races
with a global quota limit update request or with other operation that triggers
the same duplication.  These 2 races can be prevented with a ``SELECT ...  FOR
UPDATE`` on the ``quota_classes`` table.

At the time of writing, we are hoping that the bottleneck is not significant
enough to warrant the extra effort of removing it.  If time proves us wrong we
can go and implement one of these or other solution.


Changing configuration alternatives
***********************************

The `Changing configuration`_ subsection of the `Proposed change`_ section
presented the chosen mechanism to change ``quota_driver`` and
``no_snapshot_gb_quota`` configuration options, but those are not the only
possibilities.

This subsection presents 2 alternatives to make changes to the options and
ensure that **all** Cinder services run with the same configuration option
values:

- Build a complex system to orchestrate the change on running services: Signal
  the change to all Cinder services and make sure they complete ongoing quota
  operations before signaling the quota driver that it needs to do
  recalculations, then signal services that they have to reload the quota
  driver and finally continue their operation.

  Implementing this is quite complex, starting with how difficult is to make
  sure that there are no services that have missed the notification: A service
  may have a temporary loss of connection to RabbitMQ or the DB.

  We also have the difficulty of hitting pause in all running operations among
  others.

- Only allow changing the configuration option when all services are down.
  Cinder services would be smart enough to detect on start that the
  configuration has changed and confirm that they are the only service that is
  currently running and can proceed to tell the quota driver that it needs to
  do the recalculations.

  We find multiple challenges when trying to make Cinder smart enough to detect
  that there are no other services running:

  - We have no way of knowing if a Cinder API service is running or not because
    they don't issue DB heartbeats and they don't receive any RPC calls via
    RabbitMQ.  We can make them issue DB heartbeats and we may even make them
    listed on a message queue for RabbitMQ messages.

  - When starting all Cinder services at the same time they need to avoid
    racing to tell the driver to recalculate the quotas.  This can be resolved
    locking for update a DB row to prevent the race and allowing only 1 service
    to do the calculation.

  - Even if all services are brought down, the DB heartbeat for the services
    won't timeout for a while, so Cinder will have to wait until those
    heartbeats time out.  This introduces an unnecessary delay on the Cinder
    restart.

  - Spurious/temporary network issues that may make Cinder think that there are
    no running services.  This is actually the biggest issues.

Since changing the quota configuration options is not something that's going to
be frequently changed, we actually expect it to be change once at most, passing
the responsibility of doing it right to the system administrator seems like the
best choice.

In any case this is not something that has to be like that forever.  If this
behavior becomes a real problem we can change it in the future.

Data model impact
-----------------

The new quota system will place greater importance on the database queries and
reduce the Python code, so the database needs to be ready to perform counting
queries efficiently.

The main change to ensure efficiency will be adding the proper indexes to
resource tables.  The index that we'll need to have is one on the
``project_id`` and ``deleted`` columns for the following tables (that currently
don't have them):

- ``volumes``
- ``snapshots``
- ``backups``
- ``groups``

This changes will bring additional benefits to the Cinder service, because
right now listing resources on a deployment with many projects or with many
deleted resources is not efficient, because the database has to go through
*all* the resources to filter out non delete rows that belong to a specific
project (`bug #1952443 <https://bugs.launchpad.net/cinder/+bug/1952443>`_).

Existing ``quota_usages`` tables will no longer be used, and it will be removed
on the next release after the rollout of the new quota system.

The ``reservations`` table will remain in use, although only for a couple of
operations.

Additionally we'll need to track the current quota driver that is being used as
well as the ``no_snapshot_gb_quota`` configuration option to be able to tell
the quota drivers if they have changed.

For this purpose the proposal is to create a new table that can store global
cinder information.

Table ``global_data`` will have the following fields:

- ``created_at``: When this key-value pair was created

- ``updated_at``: When this key-value pair was last updated

- ``key``: String value describing the value. For example
  ``no_snapshot_gb_quota`` or ``quota_driver``.

- ``value``: String with the value of the key. For example ``true`` or
  ``StoredQuotaDriver``.

REST API impact
---------------

There will be no REST API impact, because we currently only expose the usage
and reservations (``quota_usages`` table) through a listing API that we'll
still be able to provide with the current quota driver interface, and we still
have reservations (even if fewer operations use them) so that information still
remains relevant and we don't need to remove it from the response.

Security impact
---------------

None.

Active/Active HA impact
-----------------------

None.

Notifications impact
--------------------

No notification impact, since no new operations are added or removed.

Other end user impact
---------------------

End users of the Cinder service should not have significant impact, except for
how the quota is counted.

Current behavior can be erratic on how quota is counted, as it will depend on
how and where things fail, so we can have volumes in ERROR status that have
been counted towards quota and others that have not.

With this new approach the quota consumption rules will be very
straightforward, users/admins just need to list resources and add all that have
the ``consumes_quota`` field set to ``true`` to check if usage is correct.

This stable behavior, can have a positive impact on a deployment regardless of
how services are going down, since end users will not be able to go over their
allowed quota and be forced to clean failed resources instead of being able to
leave them be.

Performance Impact
------------------

Some preliminary code was prototyped for the volume creation and get usage
operations to evaluate the performance of the different quota drivers: the old,
the new ``StoredQuotaDriver``, and the new ``DynamicQuotaDriver``.

The results showed that the new ``StoredQuotaDriver`` system was twice as fast
as the old code in both operations, and the ``DynamicQuotaDriver`` was slower
than the ``StoredQuotaDriver``, as expected, but faster than the old one until
there are around 26000 resources per project.

So the ``DynamicQuotaDriver`` is less likely to be out of sync with reality
because it doesn't store fixed values, but the ``StoredQuotaDriver`` has better
performance, and that's the reason why both drivers are going to be
implemented, to allow system administrators decide which one is better for
them.

The default driver will be ``DynamicQuotaDriver`` to prioritize the usage
values always stay in sync, and large deployments or those looking for best
performance will be able to use the ``StoredQuotaDriver``.

Deployments may even start with one quota system and then switch to the other
if necessary.

Other deployer impact
---------------------

* As soon as the new code is deployed and executed the new quota system will be
  used, there will be no backward compatibility support for old quota code.

* Deployments using a custom external quota driver will no longer be able to
  start.  This should not be a problem as we believe there is nobody using a
  custom driver.

* During rolling upgrades the quota system will be more fragile than usual, and
  users may be able to go over quota.

* New quota system will no longer have an internal brute force cleaning
  mechanism of quotas, the volume state change API will be used to clean
  reservations, and the ``cinder-manage quota sync`` command will be used for
  the ``StoredQuotaDriver``, so the following configuration options will be
  deprecated and will no longer have any effect:
  ``reservation_expire``, ``reservation_clean_interval``, ``until_refresh``,
  and ``max_age``.

* Configuration option ``use_default_quota_class`` will be deprecated, because
  all deployments will use the default quota class instead of supporting the
  already deprecated configuration file quota limits
  (`related bug #1609937 <https://bugs.launchpad.net/cinder/+bug/1609937>`_).

Developer impact
----------------

There should be a positive impact on Cinder developers, since the code should
be more readable without all the quota code in between the higher level logic,
and adding new code should not require touching the quota manually.

The new code will probably break the cinderlib project, so changes to the
project will also be necessary.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  Rajat Dhasmana (whoami-rajat)

Work Items
----------

As discussed in the PTG/mid-cycle this work may be split in 2 phases that may
be implemented in different releases:

Phase 1: ``DynamicQuotaDriver``
*******************************

- Deprecate configuration options and log warnings for deployments that are
  using custom quota drivers.

- Add required indexes to ``volumes``, ``snapshots``, ``backups``, and
  ``groups`` tables.

- Add missing ``backup`` and ``backup_gigabytes`` default quota limits to the
  ``quota_classes`` table.

- Remove deprecated ``consistencygroups`` resources from the ``quota_classes``,
  ``quotas``, ``quota_usages`` and ``reservations`` table.

- Write the ``DynamicQuotaDriver`` database quota driver.

- Make the following operations use the new quota driver:

  * Create volume
  * Delete volume
  * Manage volume
  * Extend volume
  * Retype volume
  * Transfer volume
  * Create snapshot
  * Delete snapshot
  * Manage snapshot
  * Backup create
  * Backup restore
  * Group create
  * Group delete

- Remove code for the old quota driver.

- Make the ``cinder-manage quota sync`` and ``check`` be ``noop``.

- Write the ``DynamicQuotaDriver`` database quota driver unit tests.

- Update existing unit tests.

- Write initial documentation and mention that a more efficient driver will be
  coming in the future.

Phase 2: ``StoredQuotaDriver``
******************************

- Write the ``StoredQuotaDriver`` database quota driver.

- Write the ``StoredQuotaDriver`` database quota driver unit tests.

- Update the ``cinder-manage quota sync`` and ``check`` commands.

- Add the ``cinder-manage quota change`` command.

- Do basic manual performance comparison of old and new quota system.

- Add support for the new quota system to ``cinderlib`` with a *noop* quota
  driver and use it as the default value of the ``quota_driver`` configuration
  option.

- Update documentation.

Dependencies
============

- The database engine cannot lock on non-existent rows, so the new code needs
  the database to hold default quota limit records for all the basic resources
  in the ``quota_classes`` table.  So this new code depends on us ensuring that
  the ``backups`` and ``backup_gigabytes`` records are present in the database
  and we should also have the ``consistencygroups`` removed since they haven't
  been used for a long time
  (`bug #1952420 <https://bugs.launchpad.net/cinder/+bug/1952420>`_).

Testing
=======

Besides some manual testing that will be performed to do some basic performance
comparison between the old and the new quota system, most of the testing will
be focused on the testing of the SQL queries.

Currently supported database engines are InnoDB and SQLite, and the second one
has some limitations and quirks that may make testing some of the queries
difficult or impossible, so some of the unit tests will be skipped on SQLite.

We may explore the possibility of running a tempest job that checks the quota
usage after finishing the tempest run and reports if it has become out of sync.

Documentation Impact
====================

The Cinder quota documentation will be updated to reflect how resources will be
tracked now, to contain description and use cases of the different quota
drivers, as well as the procedure to change the quota driver driver and the
``no_snapshot_gb_quota_toggled`` configuration option.

References
==========

* `In the Yoga PTG <https://etherpad.opendev.org/p/yoga-ptg-cinder#L469>`_ it
  was accepted that the quota system will be flaky during rolling upgrades.

* `WIP patch showing how the new quota system could look like
  <https://review.opendev.org/c/openstack/cinder/+/819691>`_

Related bugs:

* Backup creation quota warning in logs: `bug #1952420
  <https://bugs.launchpad.net/cinder/+bug/1952420>`_.

* Remove ``default_quota_class`` configuration option: `bug #1609937
  <https://bugs.launchpad.net/cinder/+bug/1609937>`_

* Warnings when creating volumes and snapshots with a type that doesn't have
  values in ``quota_classes``: `bug #1435807
  <https://bugs.launchpad.net/cinder/+bug/1435807>`_.

* Inneficient listing of resources: `bug #1952443
  <https://bugs.launchpad.net/cinder/+bug/1952443>`_

* Showing quotas for private types: `bug #1576717
  <https://bugs.launchpad.net/cinder/+bug/1576717>`_.

* Incorrect limit for private volume types: `bug #1952456
  <https://bugs.launchpad.net/cinder/+bug/1952456>`_

* Incorrect usage when changing ``no_snapshot_gb_quota``: `bug #1952635
  <https://bugs.launchpad.net/cinder/+bug/1952635>`_

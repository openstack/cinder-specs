..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Cinder Volume Active/Active support - Cleanup
=============================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion.

One of the reasons for this is that we have no concept of a cluster of nodes
that handle the same storage back-end, and we assume only one volume service
can access a specific storage back-end.

Given this premise, current code handles the cleanup for failed volume services
as if no other service is working with resources from his back-end, and that is
problematic when there are other volume services working with those resources,
as is the case on an Active/Active configuration.

This spec introduces a new cleanup mechanism and modifies current cleanup
mechanism so proper cleanup is done regardless of cinder configuration,
Active/Passive or Active/Active.


Problem description
===================

Current Cinder code only supports Active/Passive configurations, so the cleanup
takes that into account and cleans up resources from ongoing operations
accordingly, but that is incompatible with an Active/Active deployment.

The incompatibility comes from the fact that volume services on startup look on
the DB for resources that are in the middle of an operation and are from their
own storage back-end - detected by the ``host`` field - and proceed to clean
them up depending on the state they are in.  For example a ``downloading``
volume will be changed to ``error`` since the download was interrupted and we
cannot recover from it.

With the new job distribution mechanism the ``host`` field will contain the
host configuration of the volume service that created the resource, but that
resource may now be in use by another volume service from the same cluster, so
we cannot just rely on this ``host`` field for cleanup, as it may lead to
cleaning wrong resources or skipping the ones we should be cleaning.

When we are working with an Active/Active system we cannot just clean all
resources from our storage-backend that are in an ongoing state, since they may
be legitimate undergoing jobs being handled by other volume services.

We are going to forget for a moment how we are doing the cleanup right now and
focus on the different cleanup scenarios we have to cover.  One is when a
volume service "dies" -by that we mean that it really stops working, or it is
fenced- and failover boots another volume service to replace it as if it were
the same service -having the same ``host`` and ``cluster`` configurations-, and
the other scenario is when the service dies and no other service takes its
place, or the service that takes its place shares the ``cluster`` configuration
but has a different ``host``.

Those are the cases we have to solve to be able to support Active/Active and
Active/Pasive configurations with proper cleanups.


Use Cases
=========

Operators that have hard requirements, SLA or other reasons, to have their
cloud operational at all times or have higher throughput requirements will want
to have the possibility to configure their deployments with an Active/Active
configuration and have proper cleanup of resources when services die.


Proposed change
===============

Since checking for the status and the ``host`` field of the resource is no
longer enough to know if it needs cleanup -because the ``host`` field will be
referring to the ``host`` configuration of the volume service  that created the
resource and not the owner of the resource as explained in the `Job
Distribution`_ specs- we will create a new table to track which service from
the cluster is working on each resource.

We'll call this new table ``workers`` and it will include all resources that
are being processed with cleanable operations, and therefore would require
cleanup if the service that is doing the operation crashed.

When a cleanable job is requested by the API or any of the services -for
example a volume deletion can be requested by the API or by the c-vol service
during a migration- we will create a new row in the ``workers`` table with the
resource we are working on and who is working on it.  And once the operation
has been completed -successfully or unsuccessfully- this row will be deleted to
indicate processing has concluded and a cleanup will no longer be needed if the
service dies.

We will not be adding a row for non cleanable operations and resources that are
used in cleanable operations but won't require cleanup, as this would create a
significant increase in DB operations that would end up affecting performance
of all operations.

These ``workers`` rows serve as *flags* for the cleanup mechanism to know it
must check that resource in case of a crash and see if it needs cleanup.  There
can only exist 1 cleanable operation at a time for a given resource.

To ensure that both scenarios mentioned above are taken care of, we will have
cleanup code on cinder-volume and Scheduler services.

Cinder-volume service cleanups will be similar to the ones we currently have on
startup -``init_host`` method- but with small modifications to use the
``workers_table`` so services can tell which resources require cleanup because
they were left in the middle of an operation.  With this we take care of one of
the scenarios, but we still have to consider the case where no replacement
volume service comes up with the same ``host`` configuration, and for that we
will add a mechanism on the scheduler that will take care of requesting other
volume service from the cluster, that manage the same backend, to do the
cleaning for the fallen service.

The cleanup mechanism implemented on the scheduler will have manual and
automatic options, manual option will require the caller to specify which
services should be cleaned up using filters, and automatic operation will let
the scheduler decide which services should be cleaned up based on their status
and how long they have been down.

Automatic cleanup mechanism will consist of a periodic task that will sample
services that are down, with a frequency of ``service_down_time`` seconds, and
will proceed to clean up resources that were left by those services that are
down after ``auto_cleanup_checks`` x ``service_down_time`` seconds have passed
since the service went down.

Since we can have multiple Scheduler services and the cinder-volume service all
trying to do the cleanup simultaneously, code needs to be able handle these
situations.

On one hand, to prevent multiple Schedulers from cleaning the same services's
resources they will be reporting all automatic cleanup operations requested to
the cinder-volumes to the other Scheduler services and will ask other scheduler
services which services have already been cleaned on service start.

On the other hand, to prevent cleanup concurrency issues if a cleanup is
requested on a service that is already being cleaned up, we will issue all
cleanup operations with a timestamp indicating that only ``workers`` entries
before that should be cleaned up, so when a service starts doing the cleanup
for a resource it updates the entry an prevents additional cleanup operations
on the resource.

Row deletion operations in ``workers`` table will be a real deletions in the
DB, not soft deletes like we do for other tables, because the number of
operations, and therefore of rows, will be quite high and because we will be
setting constraints on the rows that would not hold true if we had the same
resource multiple times (there are workarounds, but it doesn't seem to be worth
it).

Since these will be big, complex changes, we will not be enabling any kind of
automatic cleanup by default, and it will need to be either enabled in the
configuration using ``auto_cleanup_enabled`` option or triggered using the
manual cleanup API -using filters- or the automatic cleanup API.

It will be possible to trigger the automatic cleanup mechanism via the API even
when it is disabled, as the disabling only prevents it from being automatically
triggered.

It is important to mention that using "reset-state" operation on any resource
will remove any existing ``workers`` table entry in the DB.

When proceeding with a cleanup we will ensure that no other service is working
on that resource (claiming the ``worker``'s entry) and that the data on the
``workers`` entry is still valid for the given resource (status matches) since
a user may have forcefully issued another action on the resource in the
meantime..


Alternatives
------------

There are multiple alternatives to proposed change, the most appealing ones
are:

- Use Tooz with a DLM that allows Leader Election to prevent more than one
  scheduler from doing cleanup of down services.  Downsides to this solution
  are considerable:

  - Increased dependency on a DLM.

  - Limiting DLM choices since now it needs to have Leader Election
    functionality.

  - We will still need to let other schedulers know when the leader does
    cleanups because when electing a new leader will need this information to
    determine if down services have already been cleaned.

- Create ``workers`` DB entries for every operation on a resource.
  Disadvantages of this alternative are:

  - Considerable performance impact.

  - Greatly increase cleanup mechanism complexity, as we would need to mark all
    entries as being processed by the service we are going to clean (this has
    its own complexity because multiple schedulers could be requesting it or a
    scheduler and the service itself), then see which of those resources would
    require cleanup according to the ``workers`` table and check if no other
    service is already working on that resource because a user decided to do a
    cleanup on his own (for example a force delete on a deleting resource) and
    if there's no other service working on the resource and the resource has a
    status that is cleanable, then do the cleanup.  Doing all this without
    races is quite complicated.

Data model impact
-----------------

Create new `workers` table with following fields:

- ``id``: To uniquely identify each entry and speed up some operations
- ``created_at``: To mark when the job was started at the API
- ``updated_at``: To mark when the job was last touched (API, SCH, VOL)
- ``deleted_at``: Will not be used
- ``resource_type``: Resource type (Volume, Backup, Snapshot...)
- ``resource_id``: UUID of the resource
- ``status``: The status that should be cleaned on service failure
- ``service_id``: service working on the resource


REST API impact
---------------

Two new admin only API endpoint will be created, ``/workers/cleanup`` and
``/workers/auto_cleanup``.

For ``/workers/cleanup`` endpoint we will be able to supply filtering
parameters, but if no arguments are provided cleanup will issue a clean message
for all services that are down.  But we can restrict which services we want to
be cleaned using parameters `service_id`, `cluster_name`, `host`, `binary`,
`disabled`.

Cleaning specific resources is also possible using `resource_type` and
`resource_id` parameters.

Cleanup cannot be triggered during a cloud upgrade, but a restarted service
will still cleanup it's own resources during an upgrade.

Both API endpoints will return a dictionary with 2 lists, one with services
that have been issued a cleanup request (`cleaning`) and another list with
services that cannot be cleaned right now because there is no alternative
service to do the cleanup in that cluster (`unavailable`), that way the caller
can know which services will be cleaned up.

Data returned for each service in the lists are `id`, `name`, and `state`
fields.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Small impact on cleanable operations since we have to use the ``workers`` table
to *flag* that we are working on the resource.

Other deployer impact
---------------------

None

Developer impact
----------------

Any developer that wants to add new resources requiring cleanup or wants add
cleanup for the status -new or existing- of an existing resource will have to
use the new mechanism to mark the resource as cleanable, add which states are
cleanable, and add the cleanup code.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  Michal Dulko (dulek)
  Anyone is welcome to help

Work Items
----------

- Make DB changes to add the new ``workers`` table.

- Implement adding rows to ``workers`` table.

- Change ``host_init`` to use an RPC call for the cleanup.

- Modify Scheduler code to do cleanups.

- Create devref explaining requirements to add cleanup resources/statuses.


Dependencies
============

`Job Distribution`_:
 - This depends on the job distribution mechanism so the cleanup can be done by
   any available service from the same cluster.

Testing
=======

Unittests for new cleanup behavior.


Documentation Impact
====================

Document new configuration option ``auto_cleanup_enabled`` and
``auto_cleanup_checks`` as well as the cleanup mechanism.

Document behavior of reset-state on Active-Active deployment.


References
==========

General Description for HA A/A: https://review.openstack.org/232599

Job Distribution for HA A/A: https://review.openstack.org/327283

.. _Job Distribution: https://review.openstack.org/327283

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Promote Replication Target (Cheesecake)
=====================================================

https://blueprints.launchpad.net/cinder/+spec/replication-backend-promotion

Problem description
===================

After failing backend A over to backend B, there is not a mechanism in
Cinder to promote backend B to the master backend in order to then replicate
to a backend C. We also lack some management commands to help rebuild after a
disaster if states get out of sync.

Current Workflow
----------------
1. Setup Replication
2. Failure Occurs
3. Fail over
4. Promote Secondary Backend

   a. Freeze backend to prevent manage operations
   b. Stop cinder volume service
   c. Update cinder.conf to have backend A replaced with B and B with C
   d. *Hack db to set backend to no longer be in 'failed-over' state*.
      This is the step this spec is concerned with. Example:
      ::

        update services set disabled=0,
                            disabled_reason=NULL,
                            replication_status='enabled',
                            active_backend_id=NULL
          where id=3;
   e. Start cinder volume service
   f. Unfreeze backend

Use Cases
=========
There was a fire in my data center and my primary backend (A) was destroyed.
Luckily, I was replicating that backend to backend (B). After failing over
to backend B and repairing the data center, we installed backend C to be a
new replication target for B.

There is also a case where, for whatever reason, the replication state gets out
of sync with reality. In a situation where the status and active backend id
need to be adjusted manually by the cloud admin while recovering from a
disaster.

Proposed change
===============

To handle the case where the Cinder DB just needs to be synchronized with the
real world we will add the following cinder manage commands to reset the active
backend for a host. Similar to reset-state for volumes this will just do DB
operations. The assumption being that the cinder.conf has already been updated,
and the volume service is stopped, in a disabled state, and probably frozen.
The manage command will verify that the service is stopped, disabled, and
frozen.

::

    cinder-manage reset-active-backend replication_status=<status> \
        <active_backend_id> <backend-host>

Equivalent to:
::

    update services set replication_status='<status>',
                        active_backend_id='<id>',
       where host='<backend-name>';

Where the defaults for `status` will be disabled and the `active_backend_id`
will be None. The target for this could also be a cluster in A-A deployments.

Note: It will be up to the Admin to re-enabled the service.

That gives us the ability to avoid having an admin go manually run DB commands,
but does *NOT* allow for doing this in an "online" recovery. The volume service
must be offline for this to work safely.

Making this change will require drivers to make adjustments to their
replication states upon initialization. As-is we don't really have much
definition around what is allowed to change in cinder.conf and how much the
driver should support for changes to replication targets and things like that.
There is sort of an implied contract that when `__init__` or `do_setup` is
called the driver *should* make their current replication state match what they
are given in the config. This is a little bit problematic today though as there
isn't a mechanism for them to update DB entries like
`replication_extended_status` or `replication_driver_data`. To solve that gap,
and allow for this new officially supported method of modifying replication
status when offline, we will introduce a new driver api method
::

  update_host_replication(replication_status, active_backend_id, volumes, groups)


This method will be called immediately following `do_setup` and will return a
model update for the service as well as a list of updates for the volumes and
groups if the driver supports replication groups.  The `group` parameter will
default to None if not defined.

The drivers will need to be able to take appropriate steps to get the system
into the desired state based on the current DB state (which was theoretically
modifed before startup by the new cinder-manage command) and the current
cinder.conf.

If not implemented things will continue to work as they do today, and require
that the admin potentially take more drastic measures to recover after
performing a failover. The goal here is to not break any existing
functionality, but to add enough infrastructure for drivers to behave better.

When we do this we might require some way of fencing to prevent multiple A-A
driver instances from doing the setup at the same time as it will more than
likely be problematic for some backends, and could risk data loss if more than
one is modifying replication states at the same time. As a simple solution we
could use a new replication_status like `updating` for the service, and only
allow them to call setup_replication if the status is not set to that. This
status can also be beneficial for an admin to know what is happening if the
process of updating via the driver takes noticeable amounts of time.

Doing it this way should also allow for later on making "online" updates where
we can utilize that same driver hook to modify replication states. This spec
and initial implementation does not aim to cover that scenario.

Alternatives
------------

We could add admin API's to Cinder. Those API's could do the DB updates and
ping the drivers. The downside is that it requires the API and volume service
to be online, which may be problematic in the scenario that you are picking up
pieces after a disaster.

Later on we can look into doing "online" promotions where the volume service
does not need to be offline. Similar code in the drivers would be required, but
the complexity gets increased rapidly by trying to support this.

There was also discussion about using new admin API's which would modify a db
state that tracks replication info. The downside to this is that we will move
into a scenario where the running config and state doesn't match the configured
state.

Following on that path of tracking replication state in the db, we could go to
the extreme and move all of the replication configuration to be done via API's.
We can then track state, and provide drivers with diffs as the state changes.
In the longer term that addresses the runtime vs config state disparity, but it
will be a significant change in workflow and deployments. Not to mention would
require somewhat major changes to drivers implementing replication.

Data model impact
-----------------

A new status for the service/cluster will be added.

REST API impact
---------------

None

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

Volume service startup will probably take a performance hit, depending on the
backend and how many replicated volumes need to be modified and updated.

Other deployer impact
---------------------

None

Developer impact
----------------

Driver maintainers will need to potentially implement this new functionality,
and be aware of the implications of how/when replication configuration and
status can be adjusted.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jbernard

Work Items
----------

* Implement cinder-manage reset-active-backend command
* Implement volume manager changes to allow for `update_host_replication` to be
  used at startup by drivers.
* Open a bug against each backend that supports replication and needs an
  update as a result of this change.

Dependencies
============
None

Testing
=======

None

Documentation Impact
====================

Documentation in the Admin guide for how to perform a backend promotion, and
updating the devref for driver developers to explain the expectations of
drivers implementing replication.

References
==========

None

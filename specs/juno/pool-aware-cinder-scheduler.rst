..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Pool-aware Scheduler Support
==========================================

https://blueprints.launchpad.net/cinder/+spec/pool-aware-cinder-scheduler

Cinder currently see each volume backend as a whole, even if the backend
is consists of several smaller pools with totally different capabilities
and capacities.  Such gap can cause strange issue - a backend appears
to have enough capacity to create a copy of a volume but the backend fails
to do so.  Extending Cinder to support storage pools within volume backend
will not only solve issues like that, it can also make Cinder scheduling
decision smarter as it now knows full set of capabilities of a backend.


Problem description
===================

Cinder has been designed to take look each volume backend as a whole since
day 1. The provisioning decisions are based on the statistics reported by
backends. Any backend is assumed to be a single discrete unit with a set
of capabilities and single capacity.  In reality this assumption is not
true for many storage providers and as their storage can be further divided
or partitioned into pools to offer completely different set of capabilities
and capacities. That is there are storage backends are a combination of
storage pools rather than one big single homogenous entity. Usually
volumes/snapshots cannot be placed across pools on such backends. As a result,
there are obvious problems when using them with current Cinder:

1. A volume/snapshot that is larger than any sub-pool of the target backend
   be schedule the backend would fail;
2. A sub pool may not have enough space to serve consecutive request (e.g.
   volume clones, snapshots) while the entire backend appears to have
   sufficient capacity;

These issues are very confusing and result in inconsistent user experience.
Therefore it is important to extend Cinder so that it is aware of storage
pools within backend and also use them as finest granularity for resource
placement.


Proposed change
===============

We would like to introduces pool-aware scheduler to address the need for
supporting multiple pools from one storage controller.

Terminology
-----------
Pool - A logical concept to describe a set of storage resource that
can be used to serve core Cinder requests, e.g. volumes/snapshots.
This notion is almost identical to Cinder Volume Backend, for it
has simliar attributes (capacity, capability).  The main difference
is Pool couldnot exist on its own, it must reside in a Volume
Backend.  One Volume Backend can have mulitple Pools but Pools
do not have sub-Pools (meaning even they have, sub-Pools do not get
to exposed to Cinder, yet).  Pool has a uniqueue name in backend
namespace, which means Volume Backend cannot have two pools using
same name.

Design
------
The workflow in this change is simple:
1) Volume Backends reports how many pools and what those pools
look like and are capable of to scheduler;
2) When request comes in, scheduler picks a pool that fits the need
most to serve the request, it passes the request to the backend
where the target pool resides in;
3) Volume driver gets the message and let the target pool to serve
the request as scheduler instructed.

To support placing resources (volume/snapshot) onto a pool, these
changes will be made to specific components of Cinder:
1. Volume Backends reporting capacity/capabilities at pool level;
2. Scheduler filtering/weighing based on pool capacity/capability
and placing volumes/snapshots to a pool of certain backend;
3. Record which pool a resource is located on a backend and passes
between scheduler and volume backend.

Alternatives
------------

Rather than changing scheduler, Navneet Singh has proposed an alterntive
change to make changes to Volume Manger/Driver code.  In his approach,
each sub-pool of a backend will be exposed as a service entity (e.g.
a greenthread inside of a python process), which listens to its own
RPC channel, reports its own stats to scheduler.

Related bp for this alternative:
https://blueprints.launchpad.net/cinder/+spec/multiple-service-pools

Data model impact
-----------------

No DB schema change involved, however, the host field of Volumes table
will now include pool information but no DB migration is needed.

Original host field of Volumes:
  HostX@BackendY

With this change:
  HostX@Backend:pool0

REST API impact
---------------

N/A

Security impact
---------------

N/A

Notifications impact
--------------------

Host attribute of volumes now includes pool information in it, consumer
of notification can now extend to extract pool information if needed.

Other end user impact
---------------------

No impact visible to end user.

Performance Impact
------------------

The size of RPC message for each volume stats report will be bigger than
before (linear to # of pools a backend has).  It should not really impact
the RPC facility in terms of performance and even if it did, pure text
compression should easily mitigate this problem.

Other deployer impact
---------------------

No special requirement for deployer to deploy new version of Cinder as
it is mostly transparent changes even to deployer.  The only visible change
is the additional pool info encoded into host attribute of volume records.

Scheduler is better to be updated/deployed *piror* volume services, but this
order is not mandatory.

Developer impact
----------------

For those volume backends would like expose internal pools to Cinder for more
flexibility, developer should update their drivers to include all sub-pool
capacities and capabilities in the volume stats it reports to scheduler.
Below is an example of new stats message:

.. code-block:: python

        {
            'volume_backend_name': 'Local iSCSI', #\
            'vendor_name': 'OpenStack',           #  backend level
            'driver_version': '1.0',              #  mandatory/fixed
            'storage_protocol': 'iSCSI',          #- stats&capabilities

            'active_volumes': 10,                 #\
            'IOPS_provisioned': 30000,            #  optional custom
            'fancy_capability_1': 'eat',          #  stats & capabilities
            'fancy_capability_2': 'drink',        #/

            'pools': [
                {'pool_name': '1st pool',         #\
                 'total_capacity_gb': 500,        #  mandatory stats for
                 'free_capacity_gb': 230,         #  pools
                 'allocated_capacity_gb': 270,    # |
                 'QoS_support': 'False',          # |
                 'reserved_percentage': 0,        #/

                 'dying_disks': 100,              #\
                 'super_hero_1': 'spider-man',    #  optional custom
                 'super_hero_2': 'flash',         #  stats & capabilities
                 'super_hero_3': 'neoncat'        #/
                 },
                {'pool_name': '2nd pool',
                 'total_capacity_gb': 1024,
                 'free_capacity_gb': 1024,
                 'allocated_capacity_gb': 0,
                 'QoS_support': 'False',
                 'reserved_percentage': 0,

                 'dying_disks': 200,
                 'super_hero_1': 'superman',
                 'super_hero_2': ' ',
                 'super_hero_2': 'Hulk',
                 }
            ]
        }


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zhiteng-huang (winston-d)

Work Items
----------

There are two parts of changes needed for this proposal: changes to Cinder
itself (scheduler, volume manager) and changes to Cinder drivers for those
backends which would like to expose pools to scheduler.

But even without Cinder drivers changes, it will work fine as usual without
problem since first part of change has taken compatibility in to account.

Dependencies
============

N/A


Testing
=======

A complete set of testing environment will need following scenarios:

1) Cinder uses backend does not support pool (only exposes single pool for
entire backend);
2) Cinder uses backend supports pools (with updated driver);
3) Cinder uses mixed backends;

Create a few volumes/snapshots on the backends prior upgrades, this is for
compatibility tests.

For each scenario, tests should be done in 3 steps:

1) Update cinder-scheduler (or cinder-volume), test create volume clones,
snapshots of existing volumes or delete existing volumes;
2) Test create new volumes;
3) Update the rest part of Cinder (if cinder-scheduler is updated in step 1,
update cinder-volume now, or vise versa), test create volume, create clones,
snapshots of existing volumes or delete existing volumes.

Documentation Impact
====================

No documentation impact for changes in Cinder itself.  But drivers changes
may introduce new configure options which leads to DocImpact.

References
==========


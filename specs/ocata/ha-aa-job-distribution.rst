..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Cinder Volume Active/Active support - Job Distribution
=============================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion.

One of the reasons for this is that we have no concept of a cluster of nodes
that handle the same storage back-end.

This spec introduces the concept of cluster to Cinder and aims to provide a way
for Cinder's API and Scheduler nodes to distribute jobs to Volume nodes on a
High Availability deployment with Active/Active configuration.


Problem description
===================

Right now cinder-volume service only accepts Active/Passive High Availability
configurations, and current job distribution mechanism does not allow having
multiple services grouped in a cluster where jobs can be queued to be processed
by any of those nodes.

Jobs are currently being distributed using a topic based message queue that is
identified by the `volume_topic`, `scheduler_topic`, or `backup_topic` prefix
joined with the host name and possibly the backend name if it's a multibackend
node like in `cinder-volume.localhost@lvm`, and that's the mechanism used to
send jobs to the Volume nodes regardless of the physical address of the node
that is going to be handling the job, allowing an easier transition on
failover.

Chosen solution must be backward compatible as well as allow the new
Active/Active configuration to effectively send jobs.

In the Active/Active configuration there can be multiple Volume services - this
is not mandatory at all times, as failures may leave us with only 1 active
service - with different `host` configuration values that can interchangeably
accept jobs that are handling the same storage backend.


Use Cases
=========

Operators that have hard requirements, SLA or other reasons, to have their
cloud operational at all times or have higher throughput requirements will want
to have the possibility to configure their deployments with an Active/Active
configuration.


Proposed change
===============

Basic mechanism
---------------

To provide a mechanism that will allow us to distribute jobs to a group of
nodes we'll add a new `cluster` configuration option that will uniquely
identify a group of Volume nodes that share the same storage backends and
therefore can accept jobs for the same volumes interchangeably.

This new configuration option, unlike the `host` option, will be allowed to
have the same value on multiple volume nodes, with the only requisite that all
nodes that share the same value must also share the same storage backends and
they must also share the same configurations.

By default `cluster` configuration option will be undefined, but when a string
value is given a new topic queue will be created on the message broker to
distribute jobs meant for that cluster in the form of
`cinder-volume.cluster@backend` similar to already existing host topic queue
`cinder-volume.host@backend`.

It is important to notice that `cluster` configuration option is not a
replacement of the `host` option as both will coexist within the service and
must exist for Active-Active configurations.

To be able to determine the topic queue where an RPC caller has to send
operations we'll add `cluster_name` field to any resource DB table that
currently has the `host` field we are using for non Active/Active
configurations.  This way we don't need to check the DB, or keep a cache in
memory, to figure out in which cluster is this service included, if it is in a
cluster at all.

Once the basic mechanism of receiving RPC calls on the cluster topic queue is
in place, operations will be incrementally moved to support Active-Active if
the resource is in a cluster, as indicated by the presence of a value in the
`cluster_name` resource field.

The reason behind this progressive approach instead of an all or nothing
approach is to reduce the possibility of adding new bugs and facilitating quick
fixes by just reverting a specific patch.

This solution makes a clear distinction between independent services and those
that belong to a cluster, and the same can be said about resources belonging to
a cluster.

To facilitate the inclusion of a service in a cluster, the volume manager will
detect when the `cluster` value has changed from being undefined to having a
value and proceed to include all existing resources in the cluster by filling
the `cluster_name` fields.

Having both message queues, one for the cluster and one for the service, could
prove useful in the future if we want to add operations that can target
specific services within a cluster.

Heartbeats
----------

With Active/Passive configurations a storage backend service is down whenever
we don't have a valid heartbeat from the service and is up if we do.  These
heartbeats are reported in the DB in ``services`` table.

On Active/Active configurations a service is down if there is no valid
heartbeat from any of the services that constitute the cluster, and it is up if
there is at least one valid heartbeat.

Services will keep reporting their heartbeats in the same way that they are
doing it now, and it will be Scheduler's job to separate between individual and
clustered services and aggregate the latter by cluster name.

As explained in `REST API impact`_ the API will be able to show cluster
information with the status -up or down- of each cluster, based on the services
that belong to it.

Disabling
---------

This new mechanism will change the "disabling working unit" from service to
cluster for services that are in a cluster.  Which means that once all
operations that go through the scheduler have been moved to support
Active-Active configurations, we won't be able to disable an individual service
belonging to a cluster and we'll have to disable the cluster itself.  For non
clustered services, disabling will work as usual.

Disabling a cluster will prevent schedulers from taking that cluster, and
therefore all its services, into consideration during filtering and weighting
and the service will still be reachable to all operations that don't go through
the scheduler.

It stands to reason that sometimes we'll need to drain nodes to remove them
from a cluster, but this spec and its implementation will not be adding any new
mechanism for that.  So existing mechanism, using SIGTERM, should be used to
perform graceful shutdown of cinder volume services.

Current graceful shutdown mechanism will make sure that no new operations are
received from the messaging queue while it waits for ongoing operations to
complete before stopping.

It is important to remember that graceful shutdown has a timeout that will
forcefully stop operations if they take longer than the configured value.
Configuration option is called `graceful_shutdown_timeout`, goes in [DEFAULT]
section and takes a default value of 60 seconds; so this should be configured
in our deployments if we think this is not long enough for our use cases.

Capabilities
------------

All Volume services periodically report their capabilities to the schedulers to
keep them updated with their stats, that way they can make informed decisions
on where to perform operations.

In a similar way to the Service state reporting we need to prevent concurrent
access to the data structure when updating this information. Fortunately for us
we are storing this information in a Python dictionary on the schedulers, and
since we are using an eventlet executor for the RPC server we don't have to
worry about using locks, the inherent behavior of the executor will prevent
concurrent access to the dictionary.  So no changes are needed there to have
exclusive access to the data structure.

Although rare, we could have a consistency problem among volume services where
different schedulers would not have the same information for a given backend.

When we had only 1 volume service reporting for each given backend this was not
a situation that could happen, since received capabilities report was always
the latest and all scheduler services were in sync.  But now that we have
multiple volume services reporting on the same backend we could receive two
reports from different volume services on the same backend and they could be
processed in different order on different schedulers, thus making us have
different data on each scheduler.

The reason why we can't assure that all schedulers will have the same
capabilities stored in their internal structures is because capabilities
reports can be processed in different order on different services.  Order is
preserved in *almost all* stages, volume services report in a specific order
and message broker preserves this order and they are even delivered in the same
order, but when each service processes them we can have greenthreads execution
in different order on different scheduler services thus ending up with
different data on each service.

This case could probably be ignored since it's very rare and differences would
be small, but in the interest of consistent of the backend capabilities on
Scheduler services, we will timestamp the capabilities on the volume services
before they are sent to the scheduler, instead of doing it on the scheduler as
we are doing now. And then we'll have schedulers drop any capabilities that are
older than the one in the data structure.

By making this change we facilitate new features related to capability
reporting, like capability caching.  Since capability gathering is usually an
expensive operation and in Active-Active configurations we'll have multiple
volume services requesting the same capabilities with the same frequency for
the same back-end, we could consider capability caching as solution to decrease
the cost of the gathering on the backend.

Alternatives
------------

One alternative to proposed job distribution would be to leave the topic queues
as they are and move the job distribution logic to the scheduler.

The scheduler would receive a job and then send it to one of the volume
services that belong to the same cluster and is not down.

This method has one problem, and that is that we could be sending a job to a
node that is down but whose heartbeat hasn't expired yet, or one that has gone
down before getting the job from the queue.  In these cases we would end up
with a job that is not being processed by anyone and we would need to either
wait for the node to go back up or the scheduler would need to retrieve that
message from the queue and send it to another active node.

An alternative to proposed heartbeats is that all services report using
`cluster@backend` instead of `host@backend` like they are doing now and as long
as we have a valid heartbeat we know that the service is up.

There are 2 reasons why I believe that sending independent heartbeats is a
superior solution, even if we need to modify the DB tables:

- Higher information granularity: We can report not only which services are
  up/down but also which nodes are up/down.

- It will help us on job cleanup of failed nodes that do not come back up.
  Although cleanup is not part of this spec, it is good to keep it in mind and
  facilitate it as much as possible.

Another alternative for the job distribution, which was the proposed solution
in previous versions of this specification, was to use `host` configuration
option as the equivalent to `cluster` grouping a new added `node` configuration
option that would serve to identify individual nodes.

Using such solution may lead to misunderstandings with the concept of hosts as
clusters, whereas using the cluster concept directly there is no such problem,
wouldn't allow a progressive solution as it was a one shot change, and we
couldn't send messages to individual volume services since we only had the host
message topic queue.

There is a series of patches showing the implementation of the
`node alternative mechanism`_ that can serve as a more detailed explanation.

Another possibility would be to allow disabling individual services within a
cluster instead of having to disable the whole cluster, and this is something
we can take up after everything else is done.  To do this we would use the
normal host message queue on the cinder-volume service to receive the
enable/disable of the cluster on the manager and that would trigger a
start/stop of the cluster topic queue.  But this is not trivial, as it requires
us to be able to stop and start the client for the cluster topic from the
cinder volume manager (it is managed at the service level) and be able to wait
for a full stop before we can accept a new enable request to start the message
client again.

Data model impact
-----------------

*Final result:*

A new ``clusters`` table will be added with the following fields:

- ``id``: Unique identifier for the cluster
- ``name``: Name of the cluster, it is used to build the topic queue in the
            same way the `host` configuration option is used.  This comes from
            the `cluster` configuration option.
- ``binary``: For now it will always be "cinder-volume" but when we add backups
              it'll also accept "cinder-backup".
- ``disabled``: To support disabling clusters.
- ``disabled_reason``: Same as in service table.
- ``race_preventer``: This field will be used to prevent potential races that
                      could happen if 2 new services are brought up at the same
                      time and both try to create the cluster entry at the same
                      time.

A ``cluster_name`` field will be added to existing ``services``, ``volumes``,
and ``consistencygroups`` tables.

REST API impact
---------------

Service listing will return ``cluster_name`` field when requested with the
appropriate microversion.

A new ``clusters`` endpoint will be added to list -detailed and summarized-,
show, and update operations with their respective policies.

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

Negligible if we implement the aggregation of the heartbeats on a SQL query
using exist instead of retrieving all heartbeats and doing the aggregation on
the scheduler.

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  Michal Dulko (dulek)
  Scott DAngelo (scottda)
  Anyone is welcome to help

Work Items
----------

- Add the new ``clusters`` table and related operations.

- Add Cluster Versioned Object.

- Modify job distribution to use new ``cluster`` configuration option.

- Update service API and add new ``clusters`` endpoint.

- Update cinder-client to support new endpoint and new field in services.

- Move operations to Active-Active.

Dependencies
============

None


Testing
=======

Unittests for new API behavior.


Documentation Impact
====================

This spec has changes to the API as well as a new configuration option that
will need to be documented.


References
==========

None


.. _`node alternative mechanism`: https://review.openstack.org/286599

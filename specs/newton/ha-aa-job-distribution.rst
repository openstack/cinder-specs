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

This spec introduces a slight modification to the `host` concept in Cinder to
include the concept of a cluster as well and provide a way for Cinder's API and
Scheduler nodes to distribute jobs to Volume nodes on a High Availability
deployment with Active/Active configuration.


Problem description
===================

Right now cinder-volume service only accepts Active/Passive High Availability
configurations, and current job distribution mechanism does not allow having
multiple hosts grouped in a cluster where jobs can be queued to be processed by
any of those nodes.

Jobs are currently being distributed using a topic based message queue that is
identified by the `volume_topic`, `scheduler_topic`, or `backup_topic` prefix
joined with the host name and possibly the backend name if it's a multibackend
node like in `cinder-volume.localhost@lvm`, and that's the mechanism used to
send jobs to the Volume nodes regardless of the physical address of the node
that is going to be handling the job, allowing an easier transition on
failover.

Chosen solution must be backward compatible as well as allow the new
Active/Active configuration to effectively send jobs.

In the Active/Active configuration there can be multiple Volume nodes - this is
not mandatory at all times, as failures may leave us with only 1 active node -
with different `host` configuration values that can interchangeably accept jobs
that are handling the same storage backend.


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
nodes we'll make a slight change in the meaning of the `host` configuration
option in Cinder.  Instead of just being "the name of this node" as it is now,
it will be the "logical host name that groups a set of - one or many -
homogeneous nodes under a common name. This host set will behave as one entity
to the callers".

That way the `host` configuration option will uniquely identify a group of
Volume nodes that share the same storage backends and therefore can accept jobs
for the same volumes interchangeably.

This means that now `host` can have the same value on multiple volume nodes,
with the only requisite that all nodes that share the same value must also
share the same storage backends and configuration.

Since now we are using `host` as a logical grouping we need some other way to
uniquely identify each node, and that's where a new configuration option called
`node_name` comes in.

By default both `node_name` and `host` will take the value of
`socket.gethostname()`, this will allow us to be upgrade compatible with
deployments that are currently using `host` field to work with unsupported
Active/Active configurations.

By using the same `host` configuration option that we were previously using we
are able to keep the messaging queue topics as they were and we ensure not only
backward compatibility but also conformity with rolling upgrades requirements.

One benefit of this solution is that it will be very easy to change a
deployment from Single node or Active/Passive to Active/Active without any
downtime. All that is needed is to configure the same `host` configuration
option that the active node has and configure storage backends in the same way,
`node_name` must be different though.

This new mechanism will promote the host "service unit" to allow grouping and
will add a new node unit that will equate to the old host unit in order to
maintain the granularity of some operations.

Heartbeats
----------

With Active/Passive configurations a storage backend service is down whenever
we don't have a valid heartbeat from the host and is up if we do.  These
heartbeats are reported in the DB in ``services`` table.

On Active/Active configurations a service is down if there is no valid
heartbeat from any of the nodes that constitute the host set, and it is up if
there is at least one valid heartbeat.

Since we are moving from a one to one relationship between hosts and services
to a many to one relationship, we will need to make some changes to the DB
services table as explained in `Data model impact`_.

Each node backend will be reporting individual heartbeats and the scheduler
will aggregate this information to know which backends are up/down grouping by
the `host` field.  This way we'll also be able to tell which specific nodes are
up/down based on their individual heartbeats.

We'll also need to update the REST API to report detailed information of the
status of the different nodes in the cluster as explained in `REST API
impact`_.

Disabling
---------

Even though we'll now have multiple nodes working under the same `host` we
won't be changing disabling behavior in any way.  Disabling a service will
still prevent schedulers from taking that service into consideration during
filtering and weighting and the service will still be reachable for all
operations that don't go through the scheduler.

It stands to reason that sometimes we'll need to drain nodes to remove them
from a service group, but this spec and its implementation will not be adding
any new mechanism for that.  So existing mechanism should be used to perform
graceful shutdown of c-vol nodes.

Current graceful shutdown mechanism will make sure that no new operations are
received from the AMQP queue while it waits for ongoing operations to complete
before stopping.

It is important to remember that graceful shutdown has a timeout that will
forcefully stop operations if they take longer than the configured value.
Configuration option is called `graceful_shutdown_timeout`, goes in [DEFAULT]
section and takes a default value of 60 seconds; so this should be configured
in our deployments if we think this is not long enough for our use cases.

Capabilities
------------

All Volume Nodes periodically report their capabilities to the schedulers to
keep them updated with their stats, that way they can make informed decisions
on where to perform operations.

In a similar way to the Service state reporting we need to prevent concurrent
access to the data structure when updating this information. Fortunately for us
we are storing this information in a Python dictionary on the schedulers, and
since we are using an eventlet executor for the RPC server we don’t have to
worry about using locks, the inherent behavior of the executor will prevent
concurrent access to the dictionary.  So no changes are needed there to have
exclusive access to the data structure.

Although rare, we could have a consistency problem among nodes where different
schedulers would not have the same information for a given backend.

When we had only 1 node reporting for each given backend this was not a
situation that could happen, since received capabilities report was always the
latest and all scheduler nodes were in sync.  But now that we have multiple
nodes reporting on the same backend we could receive two reports from different
Volume nodes on the same backend and they could be processed in different order
on different nodes, thus making as have different data on each
scheduler.

The reason why we can't assure that all schedulers will have the same
capabilities stored in their internal structures is because capabilities
reports can be processed in a different order on different nodes.  Order is
preserved in *almost all* stages, nodes report in a specific order and message
broker preserves this order and they are even delivered in the same order, but
when each node processes them we can have greenthreads execution in different
order on different nodes thus ending up with different data on each node.

This case could probably be ignored since it's very rare and differences would
be small, but in the interest of consistent of the backend capabilities on
Scheduler nodes, we will timestamp the capabilities on the Volume nodes before
they are sent to the scheduler, instead of doing it on the scheduler as we are
doing now. And then we'll have schedulers drop any capabilities that is older
than the one in the data structure.

By making this change we facilitate new features related to capability
reporting, like capability caching.  Since capability gathering is usually an
expensive operation and in Active-Active configurations we'll have multiple
nodes requesting the same capabilities with the same frequency for the same
back-end, so capability caching could be a good solution to decrease the cost
of the gathering on the backend.

Alternatives
------------

One alternative to proposed job distribution would be to leave the topic queues
as they are and move the job distribution logic to the scheduler.

The scheduler would receive a job and then send it to one of the hosts that
belong to the same cluster and is not down.

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
in previous versions of this specification, was to add a new configuration
option called ``cluster`` instead of ``node`` as we propose now.

In that case message topics would change and instead of using ``host`` for them
they would use this new ``cluster`` option that would default when undefined to
the same value as ``host``.

The main advantage of this alternative is that it's easier to understand the
concepts of a cluster comprised of hosts than a conceptual host entity composed
of interchangeable nodes.

Unfortunately this lone advantage is easily outweighed by the overwhelming
number of disadvantages that presents:

- Since we cannot rename DB fields directly with rolling upgrades we have to
  make progressive changes through 3 releases to reach desired state of
  renaming ``host`` field to ``cluster`` field in tables ``service``,
  ``consistencygroups``, ``volumes``, ``backups``.

- Renaming of rpc methods and rpc arguments will also take some releases.

- Backports will get a lot more complicated since we will have cosmetic changes
  all over the place, in API, RPC, DB, Scheduler, etc.

- This will introduce a concept that doesn't make much sense in the API and
  Scheduler nodes, since they are always grouped as a cluster of nodes and not
  as individual nodes.

- The risk of introducing new bugs with the ``host`` to ``cluster`` names in DB
  fields, variables, method names, and method arguments is high.

- Even if it's not recommended, there are people who are doing HA Active-Active
  using the host name, and adding the ``cluster`` field would create problems
  for them.

Data model impact
-----------------

*Final result:*

We need to split current ``services`` DB table (``Service`` ORM model) into 2
different tables, ``services`` and ``service_nodes`` to support individual
heartbeats for each node of a logical service host.

Modifications to ``services`` table will be:

- Removal of ``report_count`` field
- Removal of ``modified_at`` field since we can go back to using ``updated_at``
  field now that heartbeats will be reported on another table.
- Removal of all version fields ``rpc_current_version``,
  ``object_current_version`` as they will be moved to ``service_nodes`` table.

New ``service_nodes`` table will have following fields:

- ``id``: Unique identifier for the service
- ``service_id``: This will be the foreign key that will join with ``services``
  table.
- ``name``: Primary key. Same meaning as it holds now
- ``report_count``: Same meaning as it holds now
- ``rpc_version``: RPC version for the service
- ``object_version``: Versioned Objects Version for the service

*Intermediate steps:*

In order to support rolling upgrades we can't just drop fields in the same
upgrade we are moving them to another table, so we need several steps/versions
to do so.  Here's the steps that will be taken to reach desired final result
mentioned above:

In N release:

- Add ``service_nodes`` table.
- All N c-vol nodes will report heartbeats in new ``service_nodes`` table.
- To allow coexistence of M and N nodes (necessary for rolling upgrades) the
  detection of down nodes will be done by checking both the ``services`` table
  and the ``service_nodes`` table and considering a node down if both reports
  say the service is down.
- All N c-vol nodes will report their RPC and Objects version in the
  ``service_nodes`` table.
- To allow coexistence of M and N nodes RPC versions will use both tables to
  determine minimum versions that are running.

In O release:

- Service availability will only be done checking the ``service_nodes`` table.
- Minimum version detection will only be done checking the ``service_nodes``
  table.

- As a final step of the rolling upgrade we'll drop fields ``report_count``,
  ``modified_at``, ``rpc_current_version``, and ``object_current_version`` from
  ``services`` table as they have been moved to ``service_nodes`` table.

REST API impact
---------------

To report status of services we will add an optional parameter called
``node_detail`` which will report the node breakdown.

When ``node_detail`` is not set we will report exactly as we are doing now, to
be backward compatible with clients, so we'll be sending the service is up if
*any* the nodes forming the host set is up and sending that it is disabled if
the service is globally disabled or all the nodes of the service are disabled.

When ``node_detail`` is set to False, which tells us client knows of the new
API options, we'll return new fields:

- ``nodes``: Number of nodes that form the host set.
- ``down_nodes``: Number of nodes that are down in the host set.
- ``last_heartbeat``: Last heartbeat from any node

If ``node_detail`` is set to True, we'll return a field called ``nodes`` that
will contain a list of dictionaries with ``name``, ``status``, and
``heartbeat`` keys that will contain individual values for each of the nodes of
the host set.

Changes will be backward compatible with XML responses, but new functionality
will not work with XML responses.

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

- Add the new ``service_nodes`` table.

- Add `node_name` configuration option.

- Modify Scheduler code to aggregate the different heartbeats.

- Change c-vol heartbeat mechanism.

- Change API's service index response as well as the update.

- Update cinder-client to support new service listing.

- Update manage client service commands.


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

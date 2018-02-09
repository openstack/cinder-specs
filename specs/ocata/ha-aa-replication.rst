..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Cinder Volume Active/Active support - Replication
=================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

As it stands to reason replication v2.1 only works in deployment configurations
that were available and supported in Cinder at the time of its design and
implementation.

Now that we are also supporting Active-Active configurations this translates to
replication not properly working on this new supported configuration.

This spec extends replication v2.1 functionality to support Active-Active
configurations while preserving backward compatibility for non clustered
configurations.

Problem description
===================

On replication v2.1 failover is requested on a per backend basis, so when a
failover request is received by the API it is then redirected to a specific
volume service via an asynchronous RPC call using that service's topic message
queue.  Same thing happens for freeze and thaw operations.

It works when we have a one-to-one relation between volume services and storage
backends, but it doesn't when we have many-to-one relationship because the
failover RPC call will be received by only one of the services that form the
cluster for the storage backend and the others will be oblivious to this change
and will continue using the same replication site they had been using before.
This will result in some operations succeeding, those going to the service that
performed the failover, and some operations failing, since they are going to
the site that's not available.

While that's the primary issue, it's not the only one, since we also have to
track the replication status at the cluster level.

Use Cases
=========

Users want to have highly available cinder services with disaster recovery
using replication.

It is not enough that individual features will be available on their own as
they'll want to have them both at the same time; so being able to use either
Active-Active configurations without replication, or replication if not
deployed as Active-Active, is insufficient.

They could probably make it work if they stopped all but one volume services in
the cluster, issued the failover request, and once it has been completed they
brought the other services back up, but this would not be a clean approach to
the problem.

Proposed change
===============

The proposed change in its core is to divide the failover operation in the
driver into two individual operations, one that will do the side of things
related with the storage backend, for example force promoting volumes to
primary on the secondary site, and another that will make the driver perform
all the operations against the secondary storage device.

As mentioned before only one volume service will receive the request to do the
failover, so by splitting the operation the manager will be able to request the
local driver to do the first part of the failover and once that is done it will
send all volume nodes in the cluster handling that backend the signal that
the failover has been completed and that they should start pointing to the
failed over secondary site, thus solving the problem of some services not
knowing that a new site should be used.

This will also require two homonymous RPC calls to the drivers new methods in
the volume manager: ``failover`` and ``failover_completed``.

We will also add the replication information to the ``clusters`` table to track
replication at the cluster level for clustered services.

Given current use of the freeze and thaw operation there doesn't seem to be a
reason to do the same split, so these operations would be left as they are and
will only be performed by one volume service when requested.

This change will require vendors to update their drivers to support replication
on Active-Active configurations, so to avoid surprises we will be preventing
the volume service from starting in Active-Active configurations with
replication enabled on drivers that don't support the Active-Active
mechanism.

Alternatives
------------

The splitting mechanism for the ``failover_host`` method is pretty straight
forward, the only alternative to the proposed changed would be to split the
thaw and freeze operations as well.

Data model impact
-----------------

Three new fields related to the replication will be added to the ``clusters``
table.  These will be the same fields we currently have in the ``services``
table and will hold the same meaning:

- ``replication_status``: String storing the replication status for the whole
  cluster.
- ``active_backend_id``: String storing which one of the replication sites is
  currently active.
- ``frozen``: Boolean reflecting whether the cluster is frozen or not.

These fields will be kept in sync between the ``clusters`` table and the
``services`` table for consistency.

REST API impact
---------------

- A new action called ``failover`` equivalent to existing ``failover_host``
  will be added, and it will support a new ``cluster`` parameter in addition to
  the ``host`` field already available in ``failover_host``.

- Cluster listing will accept ``replication_status``, ``frozen`` and
  ``active_backend_id`` as filters,

- Cluster listing will return additional ``replication_status``, ``frozen`` and
  ``active_backend_id`` fields.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

The client will return the new fields when listing clusters using the new
microversion and new filters will also be available.

Failover for this microversion will accept the cluster parameter.

Performance Impact
------------------

The new code should have no performance impact on existing deployments since it
will only affect new Active-Active deployments.

Other deployer impact
---------------------

None.

Developer impact
----------------

Drivers that wish to support replication on Active-Active deployments will have
to implement ``failover`` and ``failover_completed`` methods as well as the
current ``failover_host`` method since it is being used for backward
compatibility with the base replication v2.1.

The easiest way to support this with minimum code would be to implement
``failover`` and ``failover_completed`` and then create ``failover_host`` based
on those:

.. code:: python

    def failover_host(self, volumes, secondary_id):
        self.failover(volumes, secondary_id)
        self.failover_completed(secondary_id)

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  None

Work Items
----------

- Change service start to use ``active_backend_id`` from the cluster or the
  service.

- Add new ``failover`` REST API

- Update list REST API method to accept new filtering fields and update the
  view to return new information.

- Update the DB model and create migration

- Update ``Cluster`` Versioned Object

- Make modifications to the manager to support the new RPC calls.

Dependencies
============

This work has no additional dependency besides the basic Active-Active
mechanism being in place, which it already is.

Testing
=======

Only unit tests will be implemented, since there is no reference driver that
implements replication and can be used at the gate.

We also lack a mechanism to actually verify that the replication is actually
working.

Documentation Impact
====================

From a documentation perspective there won't be much to document besides the
changes related to the API changes.

References
==========

- `Replication v2.1`__

__ https://specs.openstack.org/openstack/cinder-specs/specs/mitaka/cheesecake.html

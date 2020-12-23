..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
NVMe Connector Healing Agent
============================

https://blueprints.launchpad.net/cinder/+spec/nvmeof-client-raid-healing-agent

Daemon that monitors NVMe connections and MDRAID arrays created by the
NVMe connector, identifies faulted volume replicas, requests new replicas
and replaces faulted replicas with new ones.


Problem description
===================

When the NVMe connector connects a replicated volume, OpenStack will see it
as one volume, and has no way of monitoring managing and healing the replicas
in these MDRAID arrays. This agent will take care of that.

It will monitor the state of the MDRAID arrays and reconcile their physical
state on the host with expected state from the volume provisioner, replacing
broken legs.

For backend volume replicas, it's the storage array that takes care of
monitoring and replacing unhealthy replicas.

NVMe MDRAID moves the data replication responsibility from the backend to
the consumer.

Currently there's no mechanism to monitor and heal these replicated volumes.

We cannot do it on the Cinder side, because even if the Cinder driver detected
the issue and created a replacing volume, we have no mechanism to report the
connection information of the replacing volume to the consumer.

So the monitoring and healing needs to be on the volume consumer side.

This agent will also be greatly beneficial for scenarios where certain replicas
of an attached replicated volume go faulty, by notifying the volume provisioner
of the faulty devices, they can be marked as faulty to avoid using old data on
re-attachments and to replace them entirely.


Use Cases
=========

When working with replicated NVMe volumes that are attached to an instance
for a long time, one of the replicas may go faulty.
This agent will detect it and attempt to replace it (self heal the MDRAID,
without the need to detach and re-attach the volume).


Proposed change
===============

Add an "NVMe agent" class that will be initialized by the NVMe connector
during volume connection on a host.

Initializing this agent will spawn a monitoring task which will repeat
periodically. We are proposing this to be a native thread if possible,
but if necessary it can be an independent process.

First proposal was to use python Event Scheduler `sched.scheduler`, but other
alternatives, such as spawning a separate process communicated to via socket,
may be chosen instead.
One key problem that would need to be addressed by this selection is a scenario
where compute service goes down, while the VMs continue operating (and their
volumes remain attached) - we don't want to lose this agent in this case.

When initialized, the agent will read access information to the volume
provisioner from a pre-determined config file location, with vendor specific
format, the content of which should be provided there by the systems operator.

The task will monitor NVMe devices and MDRAID arrays built over them.

It will know which NVMe devices and MDRAID arrays to monitor based on metadata
from the volume provisioner (backend) - which it will have a custom interface
to.

It will notify volume provisioner if necessary of failed devices.

It will attempt to connect to new NVMe devices / replicas, replacing them
in the MDRAID.

Typical self healing flow:

1. volume replica goes faulty
2. agent notices faulty replica, reports to provisioner
3. provisioner marks replica as bad (so it wont be used later unless synced)
4. agent keeps pulling volume information from provisioner
5. certain grace period passes, agent sees no state changes of faulty replica
   from provisioner, so it sends explicit request to replace replica
6. provisioner replaces replica and updates volume information
7. agent pulls volume replica information, notices a replica has changed
8. agent carries out replica replacement

Alternatives
------------

Operator could use some own script to monitor connections and fix them manually

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Will call NVMe connector methods that do sudo executions of `nvme` and `mdadm`
This will happen in the new agent task that will be spawned from os-brick.

Active/Active HA impact
-----------------------

None

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

To allow multiple vendor implementations, the specific methods / logic for:

- probing the volume provisioner
- pulling / parsing volume metadata from provisioner
- reporting volume state changes to provisioner
- requesting provisioner to replace replica

Will need to be implemented on a per vendor basis.

The architecture is such that the agent will be a generic class that will
provide the interface, and the kioxia implementation will be the first
example of vendor-specific implementation.


Implementation
==============

Assignee(s)
-----------

Zohar Mamedov
  zoharm

Work Items
----------

NVMe connector will launch monitoring task on connect_volume if not running.

Task monitors NVMe devices and MDRAID arrays created by the connector.

When a replica goes faulty (as well as other events such as disconnects)
call interface method for notifying volume provisioner.

When replicated volume devices are changed by the volume provisioner,
reconcile the physical state of NVMe devices and MDRAID arrays on the host.


Dependencies
============

None


Testing
=======

We should be able to accept this with just unit tests.


Documentation Impact
====================

Document that using NVMe connector with replicated volumes will optionally
launch this agent.


References
==========

Architectural diagram
https://wiki.openstack.org/wiki/File:Nvme-of-add-client-raid1-detail.png

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
NVMeoF Connection Agent
=======================

https://blueprints.launchpad.net/cinder/+spec/nvmeof-connection-agent

Daemon that monitors NVMeoF connections and MDRAID arrays created by the
new NVMeoF connector. It reports initiator-side events to the storage
orchestrator, identifies faulted volume replicas, requests new replicas and
replaces faulted replicas with newly assigned ones.


Problem description
===================

When the NVMe connector connects a client-replicated volume, OpenStack will see
it as one volume, and has no way of monitoring managing and healing the
replicas in these MDRAID arrays. This agent will take care of that.

Currently there's no mechanism to monitor and heal these replicated volumes.
We cannot do it only on the Cinder driver side because currently there is no
integrated mechanism to detect initiator connection events and carry out
replica replacement on the compute node.

For target-side volume replication (traditional approach), it is the storage
backend that takes care of monitoring and self healing.
The NVMe + MDRAID approach moves the data replication responsibility from the
storage backend to the consuming initiator (ie. compute node).

So the monitoring and healing needs to be on the initiator / compute side.

With this approach, the agent will monitor the NVMeoF connections and report
changes to the storage orchestrator / provisioner. It will monitor MDRAID arrays
and reconcile their physical state on the host with expected state from the
volume provisioner, replacing broken legs.

Finally, orchestration decisions / optimizations will be carried by the volume
orchestrator / provisioner using reported information from agent monitoring.
Though this is outside the scope of the agent (it is storage backend
implemented functionality) - it is useful to mention here that it will handle
cases such as avoid using faulty replicas during re-attachment scenarios,
because in this design approach only the initiator node can detect the
replicas' sync states of its MDRAID arrays.


Use Cases
=========

When working with replicated NVMeoF volumes that are attached to an instance
for a long time, one of the replicas may go faulty.
This agent will detect it and attempt to replace it, i.e., self heal the
MDRAID array, without the need to detach and re-attach the entire volume from
the instance.

Additionally, the agent will detect and report connection and replica sync
state events to the storage orchestrator (or potentially other endpoints
that can make use of it) - which is gathered by the storage backend for making
storage provisioning / orchestration decisions, as well as for telemetry.


Proposed change
===============

Add an agent entry point code to os-brick, such as:
`os_brick/os-brick/cmd/agent.py`

Add an `entry_points` `console_scripts` entry in os-brick's `setup.cfg`

The agent main function will first initialize the agent by reading access
information to the volume orchestor / provisioner from a pre-defined config
file (such as `/etc/nvme-agent/agent.conf` ?)

Vendor specific params will be used and prefixed by the vendor prefix, such as:
`kioxia_provisioner_ip`
`kioxia_provisioner_port`
`kioxia_provisioner_token`
`kioxia_cert_file`


Once initialized, the agent will start a periodic task that will do
the following:

- Host probe / heartbeat to the storage orchestrator / provisioner
- Pull volume metadata (connection and replication state) from provisioner
- Monitor NVMeoF devices and MDRAID arrays belonging to it
- Detect connection and replication state changes and report to provisioner
- Request replacements for faulty replicas
- Reconcile replica states from provisioner (carry out the replacements)


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

Operator could use some own scripts to monitor connections and replicated
arrays states, report detected events, and carry out replica replacements
manually.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Sudo executions of `nvme` and `mdadm`
Needs access for reading of root filesystem paths such as:
`/sys/class/nvme-fabrics/...`
`/sys/class/block/...`


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

If configured to run by the operator, this will be a new process running on
the compute node. Though it will spend most of its time sleeping, it will
wake up every 30 seconds to do its periodic tasks: probe the storage provisioner
and inspect nvme connections and mdraid states.

These tasks are not compute intensive, with time mostly spent waiting for a
response from the storage provisioner, and the nvme and mdraid operations will
only have time complexity linear to the number of devices under the control of
the agent (which can be treated as constant due to a low upper limit per host).
And finally, the performance effect on the network will also be small, since it
will only be sending/receiving small amounts of (meta)data across the network.

Other deployer impact
---------------------

None

Developer impact
----------------

To allow multiple vendor implementations, the specific methods / logic for:

- probing / heartbeating the storage provisioner
- pulling / parsing volume metadata from provisioner
- reporting state changes to provisioner
- requesting provisioner to replace replica

These all involve communication with and functionality carried out by the
storage backend provisioner / orchestrator, will need to be implemented on
a per vendor basis.

The architecture is such that the agent will be a generic daemon that will
define the interface, and the kioxia implementation will be the first
example of a vendor-specific implementation.


Implementation
==============

Assignee(s)
-----------

Zohar Mamedov
  zoharm

Work Items
----------

Agent entry point and initialization.

Agent periodic tasks:

- Host probe / heart beat
- Monitoring (connection and replication event detection)
- Report events to storage provisioner
- Connection and replication state re-conciliation


Dependencies
============

None


Testing
=======

We should be able to accept this with just unit tests.


Documentation Impact
====================

Document that with this feature os-brick will be coming with a console-script
that is used to launch this agent.

Document how to configure the agent for usage.


References
==========

Presentation slides with architectural diagram on slide 2
https://docs.google.com/presentation/d/1lPU8mQ7jJmr9Tybu5gXkbE7NC1ppkMnoBS4cgSFhzWc

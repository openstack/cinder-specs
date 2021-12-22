..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
NVMeoF Multipathing
===================

https://blueprints.launchpad.net/cinder/+spec/nvmeof-multipath

Extend the NVMeoF connector to support two types of multipathing:

* Native NVMeoF multipathing
* DeviceMapper based multipathing

At the time of writing, only TCP and RDMA protocols are supported and
contemplated by the spec (so it does not contemplate the case for other
transport protocols such as FC).


Problem description
===================

Currently the NVMeoF connector only supports single pathed connections.

The problem description can be summed up by the statement:
A single pathed connection is a single point of failure.

For resiliency, and to catch up with industry standards, support for
multi pathed connections is needed, mitigating the single point of failure.

Another potential benefit is performance while maintaining replication, such
as in cases where there is synchronous replication on the backend, and the
multipath mechanism can switch on failure.

This can be done in either of two ways:

NVMeoF ANA (Asynchronous Namespace Access) is the NVMeoF native way of doing
multipathing, and it offers an interesting set of features.
A spec is available for more under-the-hood details [1]_.

Using DeviceMapper for multipathing is an industry standard approach (while
NVMeoF ANA support is not yet) with connections such as iSCSI, and can be
equally utilized with NVMeoF connections.
The goal of DM component of this spec's implementation is to bring the NVMeoF
connector up to speed with using DeviceMapper for multipathed connections.


Use Cases
=========

Whether via native multipath or DeviceMapper, the use case is the ability to
expose multipathed connections to volumes, mitigating connection single point
of failure and improving performance.

For example, an NVMeoF volume has two paths (portals) and a multipath
connection is established to it. If one of the paths breaks (such as in a
partial network failure) - the attached device would remain functional,
whereas with single pathed connections, the device would become unavailable
to the host it was attached to.


Proposed change
===============

The multipathing implementation will be broken up into its two main component
features - Native and DM multipathing:

Phase 1 (Native)
----------------

The NVMeoF connector will check the host's multipath capability and report
it in the connector properties passed to the volume driver, this will also
allow any driver / backend specific handling.

The value of `/sys/module/nvme_core/parameters/multipath` can be `Y` or `N` -
this should be returned as a True or False `nvme_native_multipath` property in
`get_connector_properties` method from the `NVMeOFConnector` connector class.

The "new" NVMeoF connector API (introduced in Wallaby) supports passing
multiple portals for the same NVMeoF device. Currently only the first
successful connection is established.

In multipath mode, the portals connection method should simply connect to
all portals by running the `nvme connect` cli command for each one.

The NVMe system under the hood will create invisible devices for each of the
portal connections, and expose a single multipath device on top of them.

It should be noted that the native multipath spec requires the target portals
to expose the device under the same target NQN and namespace id for multipath
to happen. (This is the backend's responsibility and should be transparent to
OpenStack code)

Finally, the new device will be matched by its uuid as is already implemented
in the baseline single path connector.

The connection information for a native multipath volume will look like:

.. code-block:: python

  {
      'vol_uuid': '...someuuid...',
      'target_nqn': '...somenqn...',
      'portals': [
          ('10.1.1.1', 4420, 'tcp'),
          ('10.1.1.2', 4420, 'tcp')
      ]
  }

Regarding `enforce_multipath` - it is a DeviceMapper multipathing parameter
and it will not have any interaction with the native NVMe multipathing.

Phase 2 (DeviceMapper)
----------------------

For DeviceMapper multipathing, use `dm_replicas` in the connection information
to specify all the replicas. This approach allows for multiple NQNs for one
synced volume, which is often a requirement for DM-based multipathing.

`dm_replicas` cannot be used together with `volume_replicas` which is for RAID
replication only. The drivers should know not to pass that. If both are
passed, the operation should fail.

As in the baseline connector, connect to all `dm_replicas` (it is worth
mentioning that each replica could have multiple portals, essentially allowing
this DeviceMapper aggregation to run on top of native multipath devices)

Once connections are established, DeviceMapper will handle the multipathing
under the hood (should be transparent to OpenStack code)

DeviceMapper needs to recognize that two devices should be "merged"
It does so based on device id information
Therefore, it is the responsibility of the backend to expose devices with
id information that conforms to the device matching requirements
of DeviceMapper.

Since DeviceMapper recognizes two devices should be "merged" based on device
id information (udev?) - it is the responsiblity of the backend to handle its
exposed devices to conform to the requirement.

The connection information for a DeviceMapper multipath volume will look like:

.. code-block:: python

  {
      'vol_uuid': '...someuuid...',
      'dm_replicas': [
          {
              'vol_uuid': '...someuuid1...',
              'target_nqn': '...somenqn1...',
              'portals': [('10.1.1.1', 4420, 'tcp')]
          },
          {
              'vol_uuid': '...someuuid2...',
              'target_nqn': '...somenqn2...',
              'portals': [('10.1.1.2', 4420, 'tcp')]
          }
      ]
  }


Connector API Summary
---------------------

With this proposal, there will be multiple replication and/or multipath mode
permutations of operation for the connector:
- Single pathed connection
- Native multipathed connection
- RAID connection (single or multipathed)
- DeviceMapper connection (single or native multipath)

Note: RAID on top of DM is not allowed as it is inconsistent with the
connection information API (and would need an unnecessary overcomplication of
it as well as the physical implementation). It could theoretically be added
one day based on need, but it is overkill on multiple levels.

Single path connection information for cinder driver:

.. code-block:: python

  {
      'vol_uuid': '...someuuid...',
      'target_nqn': '...somenqn...',
      'portals': [('10.1.1.1', 4420, 'tcp')]
  }

Connection information for cinder driver that supports native multipathed
connections (os-brick will iterate them if a single path connection is
requested):

.. code-block:: python

  {
      'vol_uuid': '...someuuid...',
      'target_nqn': '...somenqn...',
      'portals': [
          ('10.1.1.1', 4420, 'tcp'),
          ('10.1.1.2', 4420, 'tcp')
      ]
  }

For DeviceMapper, the example below is given with no native multipath for the
replica devices. For these replica devices to do native multipathing, portals
for each of each device's paths should be specified in the `portals` list.

Connection information for DeviceMapper:

.. code-block:: python

  {
      'vol_uuid': '...someuuid...',
      'dm_replicas': [
          {
              'vol_uuid': '...someuuid1...',
              'target_nqn': '...somenqn1...',
              'portals': [('10.1.1.1', 4420, 'tcp')]
          },
          {
              'vol_uuid': '...someuuid2...',
              'target_nqn': '...somenqn2...',
              'portals': [('10.1.1.2', 4420, 'tcp')]
          }
      ]
  }


Alternatives
============

Both alternative for NVMeoF multipathing that I am aware of are described
in this spec: Native and DeviceMapper

Data model impact
=================

None

REST API impact
===============

None

Security impact
===============

Same as the baseline NVMeoF connector:

* Sudo executions of `nvme`
* | Needs access for reading of root filesystem paths such as:
  | `/sys/class/nvme-fabrics/...`
  | `/sys/class/block/...`


Active/Active HA impact
=======================

None

Notifications impact
====================

None

Other end user impact
=====================

None

Performance Impact
==================

The main benefit is of increased I/O while maintaining replication - such as
in a synchronous replication active-passive failover multipath configuration.

One performance impact should happen during attachments on the host, where
now multiple NVMeoF devices (rather than just one) would need to be connected,
and a new device exposed on top of them.

Depending on the mode of multipath connection (for example active-active vs
active-passive) - there could be an increase in network resource usage for
the multipath device io.


Other deployer impact
=====================

None

Developer impact
================

Driver developers that want to use NVMeoF will need to be aware of the
connector API - specifically the connection_information spec described

It is also worth mentioning that storage backends (although transparent to
OpenStack code) will need to keep in mind the system level requirements
for whichever multipath mode they use (Native or DeviceMapper)


Implementation
==============

Assignee(s)
-----------

* Zohar Mamedov (zoharm)
* Anyone Else

Work Items
----------

Phase 1 (Native):

* Connector properties check kernel supports native multipath

* Establish all (multiple) necessary connections to multipath volume and ensure
  proper device is exposed.


Phase 2 (DeviceMapper):

* Ensure all sync replicated devices are connected.

* Ensure the proper DeviceMapper device is exposed.


Dependencies
============

Baseline dependencies are same as baseline single pathed connector:
relevant nvmeof kernel modules, and nvme-cli

For native NVMe multipathing, the kernel needs to support it

For DeviceMapper multipathing, DeviceMapper needs to be operational


Testing
=======

Unit tests.
Also there is an upcoming gate job that will use LVM to test NVMeoF (where
multipath testing for it can be included there)


Documentation Impact
====================

Document the new connection_information API described in this spec - especially
for drivers to be able to follow and use it.


References
==========

.. [1] ANA Spec - Section 8.1.3: https://nvmexpress.org/wp-content/uploads/NVMe-NVM-Express-2.0a-2021.07.26-Ratified.pdf

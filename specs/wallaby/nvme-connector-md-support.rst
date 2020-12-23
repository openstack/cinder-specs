..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
NVMe Connector Support MDRAID replication
==========================================

https://blueprints.launchpad.net/cinder/+spec/nvme-of-add-client-raid-1

NVMe connector replicated volume support via MDRAID.
Allow OpenStack to use replicated NVMe volumes that are distributed on
scale-out storage.


Problem description
===================

When consuming block storage, resilience of the block volumes is required.
This can be achieved on the storage backend with NVMe and iSCSI though the
target will remain a single point of failure. And with multipathing, the HA
of paths to a target does not handle volume data replication.

A storage solution exposing high performance NVMeoF storage would need a
way to handle volume replication. We propose achieving this by using RAID1
on the host, since the relative performance impact will be less significant
in such an environment and will be greatly outweighed by the benefits.


Use Cases
=========

When resiliency is needed for NVMeoF storage.

Taking advantage of NVMe over a high performance fabric, gain benefits of
replication on the host while maintaining good performance.

In this case volume replica failures will have no impact on the consumer,
and will allow for self healing to take place seamlessly.

For developers of volume drivers, this will allow support for OpenStack
to use replicated volumes that they would expose in their storage backend.

End users operating in this mode will benefit from performance of NVMe with
added resilience due to volume replication.


Proposed change
===============

Expand the NVMeoF connector in os-brick to be able to take in connection
information for replicated volumes.

When `connect_volume` is called with replicated volume information, NVMe
connect to all replica targets and create a RAID1 over the devices.

.. image:: https://wiki.openstack.org/w/images/a/ab/Nvme-of-add-client-raid1-detail.png
   :width: 448
   :alt: NVMe client RAID diagram

Alternatives
------------

Replication can also be actively maintained on the backend side, which is
the way it is commonly done. However, with NVMe on a high performance fabric,
the benefits of handling replication on the consumer host can outweigh the
performance costs.

Adding support for this also increases the type of backends we fully support,
as not all vendors will chose supporting replication on the backend side.

And we can support both without any kind of impact on each one.

Data model impact
-----------------

NVMe Connector's `connect_volume` and `disconnect_volume` param
`connection_properties` can now also hold a `volume_replicas` list, which will
contain necessary info for connecting to and identifying the NVMe subsystems
for then doing MDRAID replication over them.

Each replica dict in `volume_replicas` list must include the normal flat
connection properties.

If `volume_replicas` has only one replica, treat it as non-replicated volume.

If `volume_replicas` is ommitted, use the normal flat connection properties
for NVMe as in the existing version of this NVMe connector.

non-replicated volume connection properties::

    {'uuid': '96e25fb4-9f91-4c88-ab59-275fd354777e',
     'nqn': 'nqn.2014-08.org.nvmexpress:uuid:...'
     'portals': [{
         'address': '10.0.0.101',
         'port': '4420',
         'transport': 'tcp'
     }],
     'volume_replicas': None}

replicated volume connection properties::

    {'volume_replicas': [
         {'uuid': '96e25fb4-9f91-4c88-ab59-275fd354777e',
          'nqn': 'nqn.2014-08.org.nvmexpress:uuid:...'
          'portals': [{
              'address': '10.0.0.101',
              'port': '4420',
              'transport': 'tcp'
          }]},
         {'uuid': '12345fb4-9f91-4c88-ab59-275fd3544321',
          'nqn': 'nqn.2014-08.org.nvmexpress:uuid:...'
          'portals': [{
              'address': '10.0.0.110',
              'port': '4420',
              'transport': 'tcp'
          }]},
    ]}

REST API impact
---------------

None

Security impact
---------------

Requires elevated priviliges for managing MDRAID.
(Current NVMe connector already does sudo executions of nvme cli, so this
change will just add execution of `mdadm`)


Active/Active HA impact
-----------------------

None


Notifications impact
--------------------

None

Other end user impact
---------------------

Working in this replicated mode will allow for special case scenarios where for
example a MDRAID array with 4 replicas loses connection to two of the replicas,
keeps writing data to two remaining ones. Then, after a re-attach from a reboot
or a migration, for some reason now has access to only the two originally
lost replicas, and not the two "good" ones, then the re-created MDRAID array
will have old / bad data.

The above can be remedied by storage backend awareness of devices going faulty
in the array. This is enabled by the NVMe monitoring agent, which can recognize
replicas going faulty in an array and notify the storage backend, which will
mark these replicas as faulty for the replicated volume.

Multi attach is not supported for NVMe MDRAID volumes.

Performance Impact
------------------

Replicated volume attachments will be slower (need to build MDRAID array).
It's a fair tradeoff, slower attachment for more resiliency.

Other deployer impact
---------------------

NVMe and MDRAID and their CLI clients (`nvme` and `mdadm`) need to be
available on the hosts for NVMe connections and RAID replication respectively.

Developer impact
----------------

Gives option for storage vendors to support replicated NVMeoF volumes via
their driver.

To use this feature, volume drivers will need to expose NVMe storage that is
replicated and provide necessary connection information for it when using this
feature of the connector.

This would not affect non-replicated volumes.


Implementation
==============

Assignee(s)
-----------

Zohar Mamedov
  zoharm

Work Items
----------

All done in NVMe connector:

- In `connect_volume` parse connection information for replicated volumes.
- Connect to NVMeoF targets and identify the devices.
- Create MD RAID1 array over devices.
- Return symlink to MDRAID device.
- `disconnect_volume` destroy the MDRAID.
- `extend_volume` grow the MDRAID.


Dependencies
============

NVMe and MDRAID and their CLI clients (`nvme` and `mdadm`) need to be
available on the hosts for NVMe connections and RAID replication respectively.
Fail gracefully if they are not found.

Testing
=======

In order to properly test this in tempest, programmatic access will be needed
to the storage backend. For example, to fail one of the drives of a replicated
volume.

We could also slide by with just a check of connected NVMe subsystems
(`nvme list`) and scan of MDRAID arrays (`mdadm -D scan`) to see that multiple
NVMe devices were connected and a RAID was created.

In either case tempest will need to be aware that the storage backend is
configured to use replicated NVMe volumes and only then do these checks.

Aside from that, running tempest with NVMe replicated volume backend
will still fully test this functionality, it's just the specific assertions
(nvme devices and RAID info) that would be different.

Finally, we will start with unit tests, and have functional tests in os-brick
as a stretch goal.

Documentation Impact
====================

Document that NVMe connector will now support replicated volumes and that
connection information of replicas is required from the volume driver to
support it.


References
==========

Architectural diagram
https://wiki.openstack.org/wiki/File:Nvme-of-add-client-raid1-detail.png

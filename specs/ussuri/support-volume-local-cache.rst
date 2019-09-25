..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Support volume local cache
==========================

https://blueprints.launchpad.net/cinder/+spec/support-volume-local-cache

This blueprint proposes to add support of volume local cache in cinder and
os-brick. Cache software such as open-cas [5]_ can use fast NVME SSD or
persistent memory (configured as block device mode) to cache for slow remote
volumes.

Problem description
===================

Currently there are different types of fast NVME SSDs, such as Intel Optane
SSD, with latency as low as 10 us. What's more, persistent memory which aim to
be SSD size but DRAM speed gets popupar now. Typical latency of persistent
memory would be as low as hundreds of nanoseconds. While typical latency of
remote volume for a VM can be at the millisecond level (iscsi / rbd). So these
fast SSDs or persistent memory can be mounted locally on compute nodes and used
as a cache for remote volumes.

In order to do the cache, three projects need to be changed. These are cinder,
os-brick, and Nova. Meanwhile related hardware should be added in system, e.g.
high performance SSD or persistent memory. The mechanism is similar to volume
encryption where dm-crypt is used. Cache software support would be added in
os-brick and Nova will call os-brick when it is trying to attach a volume,
os-brick then calls the cache software to setup the cache for the volume. After
that, a new virtual block device would be created and laying upon the original
block device. os-brick will expose this new virtual block device to Nova,
meanwhile the original block device mount point will remain unchanged. All of
the cache setup and teardown would be handled within os-brick.

A property would be added in the extra specs of the volume type. If a volume
type has extra-spec of "cacheable", then it means the related volumes can be
cached in compute node locally.

Like all the local cache solution, multi-attach cannot be supported. This is
because cache on node1 doesn't know the changes made to backend volume by
node2. So cinder should guarantee these two properties cannot be set at the
same time.

Considering VM migration, cache software should not format the source volume
(the volume to be cached). So cache software such as bcache cannot be used.
This spec only supports open-cas. open-cas is easy to use, you just need to
specify a block device as the cache device, and then can use this device to
cache other block devices. This is transparent to upper layer and lower layer.
Regarding upper layer, guest doesn't know it is using an emulated block device.
Regarding lower layer, backend volume doesn't know it is cached, and the data
in backend volume will not have extra change because of cache. That means even
if the cache is lost for some reason, the backend volume can be mounted to
other places and become available immediately. open-cas supports the cache
modes below:

- Write-Through (WT): write to cache and backend storage at the same time. So
  the data is fully synced in cache and backend storage. There will not be any
  data corruption when cache becomes invalid.

- Write-Around (WA): like Write-Through, but only cache for volume blocks that
  are already in cache.

- Write-Invalidate (WI): write to backend storage and invalidate cache

- Write-Back (WB): write to cache and lazy write to backend storage. This mode
  has better write performance but is possible to lose data because the latest
  data is in cache, but may not be flushed to the backend storage.

- Write-Only (WO): like write-back, but only cache write, will not cache read.
  So this mode also has the possibility to lose data.

The first three modes are suitable for scenarios that data integrity must be
guaranteed, every write io is both written to cache and backend storage. In
these modes it is just like read-only cache feature in ceph. No operation would
be blocked. e.g. VM live migration, snapshot taking, volume backup and others
can be done as usual. Cache software can also be changed to others rather than
open-cas at any time because there's no dirty data in cache at any time. This
is suitable for read intensive scenario, but it will be no benefit for write io
because every write io need to go to backend storage everytime for data
integrity.

The last two modes are suitable for scenarios that have high requirements for
both read/write performance, but low requirements for data integrity. E.g. some
test environment. In these two modes, all operations that depend on the backend
volume containing full data would not work safely. So VM live migration,
snapshot, volume backups, consistency groups, etc, cannot works safely because
there may be still dirty data in cache. At least, operator need to stop the
disk io by some way and flush the cache(via casadm) before doing these
operations. Cache software also cannot be changed to others except all dirty
data has been flushed to backend.

Cache mode can be got from cache instance ID via cache admin tool. Nova passes
available cache instances ID list to os-brick, so os-brick knows the cache mode
of the cache instances. This spec would make os-brick refuse to attach volume
with last two cache modes. But cache mode is set outside of OpenStack, Cinder /
os-brick cannot control cache mode being modified after volume attached. But it
would be well documented that Write-Back (WB) and Write-Only (WO) mode is
dangerous and operators should fully know what they are doing when trying to
set to these cache modes.

Some storage backends may support discard/TRIM functionality, but cache
software doesn't have sense of this. Cache software evicts data based on policy
like 'lru'(this is the default policy of open-cas).

Some storage client, compute node, may enable multipath for volumes, e.g.
iscsi/fc volumes. Volume local cache would not work with multipath, with the
same reason of multi-attach. os-brick detects multipath via function
get_volume_paths() and would not set cache for volume with multipath.

No restrictions would be introduced on retyping a 'cacheable' volume. This is
because retyping includes two steps: 1) attach new volume 2) detach old volume.
So old volume would be released from caching, new volume would be cached.

This is the shared cache, and the number of volumes to be cached is unlimited.
A volume be cached does not mean it will occupy space in cache. The volumes
with hot IO will consume more cache space, meanwhile volumes with no IO will
not occupy any space in cache devce. The data in cache would be evicted when
the data getting cold or other volume's IO getting hot.

Use Cases
=========

* In read intensive scenarios, e.g. AI training, VM volume local cache will
  significantly boost the storage performance (throughput, and especially disk
  io latency)

Proposed change
===============

In order to do volume local cache:

- In Nova, end user selects a flavor that is advertised as having a
  volume-local-cache so guest can be landed in a server with cache capability.
  The end user should expect different performance based on what server flavor
  was chosen. More information such as error handling, user message, etc, are
  out of scope of this spec and would be defined in Nova spec [4]_.

- In Cinder, volume type that is 'cacheable' should be selected.

It is Cinder which determines and sets the 'cacheable' property. A volume
marked as "cacheable" doesn't mean it must be cached, it just means it is
eligible to be cached. Nova calls os-brick to set cache for the volume only
when the volume has the property of 'cacheable'.

Cache mode is bound to cache instance. So different cache instances can have
different cache modes. All cached volumes share same cache mode if they are
cached by the same cache instance. The operator can change cache mode
dynamically, using cache software management tool. So cache mode setting is out
of OpenStack and is not controlled by OpenStack. Operator should not change
cache to unsafe mode. os-brick just accepts the cache name and cache instance
IDs from Nova.

cache_name identifies which cache software to use, currently it only supports
'opencas'. Nova knows what cache name is, based on which cache software is
enabled in compute node.

Each compute node can have more than one cache instances. os-brick can weight
each cache instance passed in, by e.g. total cache size, how many free space,
etc, via cache admin tool(casadm), and select the best one.

Some storage types support "extend volume" which triggered from cinder side.
e.g. via command "cinder extend ...". It works normally when the volume is not
"in-use". But if the volume is attached and "in-use", os-brick would not
support to extend and just raise NotImplementedError for the volume with
cacheable volume_type. This is because open-cas don't support volume extending
dynamically in current release. But "Resize Instance" feature which triggered
from Nova still can work, because volume will be detached and then re-attached
during ""Resize Instance".

The final solution would be like::

                        Compute Node

 +---------------------------------------------------------+
 |                                                         |
 |                        +-----+    +-----+    +-----+    |
 |                        | VM1 |    | VM2 |    | VMn |    |
 |                        +--+--+    +--+--+    +-----+    |
 |                           |          |                  |
 +---------------------------------------------------------+
 |                           |          |                  |
 | +---------+         +-----+----------+-------------+    |
 | |  Nova   |         |          QEMU Virtio         |    |
 | +-+-------+         +-----+----------+----------+--+    |
 |   |                       |          |          |       |
 |   | attach/detach         |          |          |       |
 |   |                 +-----+----------+------+   |       |
 | +-+-------+         | /dev/cas1  /dev/cas2  |   |       |
 | | osbrick +---------+                       |   |       |
 | +---------+ casadm  |        open cas       |   |       |
 |                     +-+---+----------+------+   |       |
 |                       |   |          |          |       |
 |                       |   |          |          |       |         Storage
 |              +--------+   |          |    +-----+----+  | rbd   +---------+
 |              |            |          |    | /dev/sdd +----------+  Vol1   |
 |              |            |          |    +----------+  |       +---------+
 |        +-----+-----+      |          |                  |       |  Vol2   |
 |        | Fast SSD  |      |    +-----+----+   iscsi/fc/...      +---------+
 |        +-----------+      |    | /dev/sdc +-------------+-------+  Vol3   |
 |                           |    +----------+             |       +---------+
 |                           |                             |       |  Vol4   |
 |                     +-----+----+    iscsi/fc/...        |       +---------+
 |                     | /dev/sdb +--------------------------------+  Vol5   |
 |                     +----------+                        |       +---------+
 |                                                         |       |  .....  |
 +---------------------------------------------------------+       +---------+


Changes would include:

- Add "cacheable" property in extra-spec of volume type

  * Volume local cache cannot work with multiattach. So when adding "cacheable"
    to extra-spec, cinder should check if "multiattach" property exists or not.
    If "multiattach" exists, then cinder should refuse to add "cacheable"
    property to volume type, and vice versa.

  * Fill "cacheable" property in connection_info. So os-brick can know whether
    a volume can be cached or not.

- Add a common framework for different cache software in os-brick. This
  framework should be flexible to support different cache software.

  1) A base class - CacheManager would be added and the main functions would
  be:

  * __init__()

    This function would accept the parameters from Nova. Parameters include:

    root_helper - used for cache software management tools.

    connection_info - containing device path

    cache_name - specify the cache software name, currently only support 'opencas'

    instance_ids - specify cache instances that can be used. os-brick chooses a
    best one among these instances

  * attach_volume()

    This function would be called by Nova (in function _connect_volume) to
    setup cache for a volume when it is trying to attach the volume.

  * detach_volume()

    This function would be called by Nova (in function _disconnect_volume) to
    release the cache when it is trying to detach a volume.

  2) In __init__.py, a map of cache software and its python class would be
  added. So os-brick can find the correct class based on cache name.

  CACHE_NAME_TO_CLASS_MAP = {
      "opencas": 'os_brick.caches.opencas.OpenCASEngine',
      ...

  }

  Meanwhile a function like _get_engine() would be added to go through the
  map to find the correct class.

- Add the support for open-cas in os-brick.

  Implement functions attach_volume/detach_volume for open-cas.


Code work flow would be like::

             Nova                                        osbrick

                                               +
          +                                    |
          |                                    |
          v                                    |
    attach_volume                              |
          +                                    |
          |                                    |
          +                                    |
        attach_cache                           |
              +                                |
              |                                |
              +                                |
  +-------+ volume_with_cache_property?        |
  |               +                            |
  | No            | Yes                        |
  |               +                            |
  |     +--+Host_with_cache_capability?        |
  |     |         +                            |
  |     | No      | Yes                        |
  |     |         |                            |
  |     |         +-----------------------------> attach_volume
  |     |                                      |        +
  |     |                                      |        |
  |     |                                      |        +
  |     |                                      |      set_cache_via_casadm
  |     |                                      |        +
  |     |                                      |        |
  |     |                                      |        +
  |     |                                      |      return emulated_dev_path
  |     |                                      |        +
  |     |                                      |        |
  |     |         +-------------------------------------+
  |     |         |                            |
  |     |         v                            |
  |     |   replace_device_path                |
  |     |         +                            |
  |     |         |                            |
  v     v         v                            |
                                               |
 attach_encryptor and                          |
 rest of attach_volume                         +


* Volume local cache lays upon encryptor would have better performance, but
  expose decrypted data in cache device. So based on security consideration,
  cache should lay under encryptor in Nova implementation.

Alternatives
------------

* Assign local SSD to a specific VM. VM can then use bcache internally against
  the ephemeral disk to cache their volume if they want.

  The drawbacks may include:

  - Can only accelerate one VM. The fast SSD capability cannot be shared by
    other VMs. Unlike RAM, SSD normally is in TB level and large enough to
    cache for all the VMs in one node.

  - The owner of the VM should setup cache explicitly. But not all the VM
    owners want to do this, and not all the VM owners have the knowledge to do
    this. But they for sure want that the volume performance to be better by
    default.

* Create a dedicated cache cluster. Mount all the cache (NVME SSD) in the cache
  cluster as a big cache pool. Then allocate a certain amount of cache to a
  specific volume. The allocated cache can be mounted on compute node through
  NVMEoF protocol. Then use cache software to do the same cache.

  But this would be the compete between local PCIe and remote network. The
  disadvantage of doing it like this is: the network of the storage server
  would be a bottleneck.

  - Latency: Storage cluster typically provides volumes through iscsi/fc
    protocol, or through librbd if ceph is used. The latency would be at the
    millisecond level. Even with NVME over TCP, the latency would be hundreds
    of microseconds, depending on the network topology. In contrast, the
    latency of NVME SSD would be around 10 us, take Intel Optane SSD p4800x as
    example.

* Cache can be added in backend storage side, e.g. in ceph. Storage server
  normally has its own cache mechanism, e.g. using memory as cache, or using
  NVME SSD as cache.

  Similiar with above solution, latency is the disadvantage.


REST API impact
---------------

None

Data model impact
-----------------

None

Security impact
---------------

* Cache software will remove the cached volume data from cache device when
  volume is detached. But normally it would not erase the related sectors in
  cache device. So in theory the volume data is still in cache device before it
  is overwritten. Unless the cache device is plugged out, otherwise it is
  acceptable because the volume itself is also mounted and visible on host OS.
  Volume with encryption doesn't have this issue if encryption laying upon
  volume local cache.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

* Latency of VM volume would be reduced

Other deployer impact
---------------------

* Need to configure cache software in compute node

Developer impact
----------------

* The support for other cache software can be added by other developers later

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Liang Fang <liang.a.fang@intel.com>

Work Items
----------

* Implement a common framework for supporting different cache software
* Support open-cas
* Unit test be added

Dependencies
============

None

Testing
=======

* Unit-tests, tempest and other related tests will be implemented.

* Test case in particular: leverage DRAM to simulate fast ssd, act as the cache
  for open-cas; Use fio to do the 4k block size rand read test; Compare the
  result of volume with / without cache. The expected behavior is: cached
  volume would get lower latency.

Documentation Impact
====================

* Documentation will be needed. User documentation on how to use cache software
  to cache volume.

References
==========

.. [1] https://review.opendev.org/#/c/663549/
.. [2] https://review.opendev.org/#/c/663542/
.. [3] https://review.opendev.org/#/c/700799/
.. [4] https://review.opendev.org/#/c/689070/
.. [5] https://open-cas.github.io/

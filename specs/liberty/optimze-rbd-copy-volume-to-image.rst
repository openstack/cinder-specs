..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================================
Offload rbd's copy_volume_to_image function from host to ceph cluster
=====================================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/optimze-rbd-copy-volume-to-image

Offload rbd's copy_volume_to_image function from cinder-volume host to
ceph cluster if cinder volume back end and glance storage back end are in
the same ceph storage cluster.

Problem description
===================

Suppose cinder volume back end and glance storage back end are in the same
ceph storage cluster.
Suppose the volume's capacity and data size in the following description
are 1G.
Suppose cinder volume back end use "volumes" pool, and glance storage back end
use "images" pool.

Currently upload-to-image routine works as follows:
1. cinder use command "rbd export" to export the rbd image as a local file in
cinder-volume host. Read 1G, and Write 1G.
2. cinder call glance upload/update API to upload the exported local file to
glance storage back end. Read 1G, and write 1G.

So when we upload a 1G volume to image, we would read 2G data and write 2G
data. If we offload this function from cinder-volume host to ceph storage
cluster, we may only need to read 1G data and write 1G data, which could
reduce the half amount of data transmission compared to the current routine.

Benefits:

* Reduce the read/write cost of rbd's copy_volume_to_image function,
  especially when the volume's capacity and data size are big.
* Reduce the IO pressure on cinder-volume host when doing the
  copy_volume_to_image operation.

Use Cases
=========

Proposed change
===============

By offload rbd's copy_volume_to_image function from host to ceph cluster
, we need to do the following changes.

* Check whether cinder volume back end and glance storage back end are in the
  same ceph storage cluster or not.

* If not, keep the current work routine not change.

* If yes, use rbd.Image.copy(dest_ioctx, dest_name) function to copy volume
  from one pool to another pool because cinder volume back end and glance
  storage back end always use different ceph pools.

Alternatives
------------

Solution A: use rbd.Image.copy(dest_ioctx, dest_name) function to offload
rbd's copy_volume_to_image function from host to ceph cluster.

Solution B: use ceph's functions of clone image and flatten clone image to
offload rbd's copy_volume_to_image function from host to ceph cluster. It
contains the following steps.
* Create a volume snapshot snap_a and protect the snapshot snap_a.
* Clone a child image image_a of snap_a.
* Flatten the child image image_a, and snap_a has not been depended on.
* Unprotect the snapshot snap_a and delete it.

Using a volume created based on "cirros-0.3.0-x86_64-disk" to show
the time-consuming between solution A and solution B.
* [Common]Create a volume based on image "cirros-0.3.0-x86_64-disk".

.. code-block:: bash

    root@devaio:/home# cinder list
    +--------------------------------------+-----------+------+------+
    |                  ID                  |   Status  | Name | Size |
    +--------------------------------------+-----------+------+------+
    | 3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8 | available | None |  1   |
    +--------------------------------------+-----------+------+------+

* [Solution-A-step-1]Copy
  volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8 from "volumes" pool to
  "images" pool.

.. code-block:: bash

    root@devaio:/home# time rbd cp
    volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8 images/test

    Image copy: 100% complete...done.

    real    0m3.687s
    user    0m1.136s
    sys     0m0.557s

* [Solution-B-step-1]Create a snapshot of volume
  3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8 and protect it.

.. code-block:: bash

    root@devaio:/home# time rbd snap create --pool volumes --image
    volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8 --snap image_test

    real    0m3.152s
    user    0m0.018s
    sys     0m0.013s

    root@devaio:/home# time rbd snap protect
    volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test

    real    0m3.043s
    user    0m0.016s
    sys     0m0.012s

* [Solution-B-step-2]Do clone operation on
  volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test.

.. code-block:: bash

    root@devaio:/home# time rbd clone
    volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test
    images/snapshot_clone_image_test

    real    0m0.102s
    user    0m0.020s
    sys     0m0.016s

* [Solution-B-step-3]Flatten the clone image images/snapshot_clone_image_test.

.. code-block:: bash

    root@devaio:/home# time rbd flatten images/snapshot_clone_image_test

    Image flatten: 100% complete...done.

    real    0m10.228s
    user    0m1.375s
    sys     0m0.443s

* [Solution-B-step-4]Unprotect the snap
  volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test.

.. code-block:: bash

    root@devaio:/home# time rbd snap unprotect
    volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test

    real    0m0.064s
    user    0m0.019s
    sys     0m0.015s

* [Solution-B-step-5]Delete the no dependency snap
  volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test.

.. code-block:: bash

    root@devaio:/home# time rbd snap rm
    volumes/volume-3d6e5781-e3ac-4106-bfed-0aa0dd3af1f8@image_test

    real    0m0.235s
    user    0m0.017s
    sys     0m0.013s

By the above test result of solution A and solution B, solution A needs
(real:0m3.687s, user:0m1.136s, sys:0m0.557s) to finish the volume copy
operation and solution B needs (real:0m16.824s, user:0m1.465s, sys:0m0.512s)
to do that. So using solution A to offload rbd's copy_volume_to_image function
from host to ceph cluster.

Data model impact
-----------------

None

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

Offload rbd's copy_volume_to_image function from host to ceph cluster could
make full use of ceph's inherent data copy feature and the hardware capacity
of ceph storage cluster to expedite the volume data copy speed, reduce the
amount of data transmission and reduce the IO load on cinder-volume host.

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
  ling-yun<zengyunling@huawei.com>


Work Items
----------

* Implement code that mentioned in "Proposed change".

Dependencies
============

Cinder volume back end and glance storage back end are in the same
ceph storage cluster.

Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change" and ensure that Cinder copy volume to image
feature works well while introducing the feature of offload rbd's
copy_volume_to_image function from host to ceph cluster.

Documentation Impact
====================

None

References
==========

None

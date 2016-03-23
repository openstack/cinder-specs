..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Support volume migration in Ceph driver
==============================================

https://blueprints.launchpad.net/cinder/+spec/ceph-volume-migrate

Currently cinder supports volume migration with backend assistance.
Some backends like IPSANs support migrating volume in backend level, but
Ceph does not. Though Ceph volume migration has already been supported through
cinder generic migration process in Liberty, the migration efficiency is not
so good. It is necessary to support volume migration in the Ceph driver to
improve the efficiency of migration.

Problem description
===================

Suppose there are two cinder volume backends which are in the same Ceph
storage cluster.

Now volume migration between two Ceph volume backends is implemented by
generic migration logic. Migration operation can proceed with file operations
on handles returned from the os-brick connectors, but the migration efficiency
is limited on file I/O speed. If we offload migration operation from
cinder-volume host to Ceph storage cluster, we would get the following
benefits:

* Improve the volume migration efficiency between two Ceph storage pools,
  especially when the volume's capacity and data size is big.
* Reduce the IO pressure on cinder-volume host when doing the volume
  migration.

Use Cases
=========

There are three cases for volume migration. The scope of this spec is for
the available volumes only and targets to resolve the issues within
the following migration case 1:

Within the scope of this spec:

  1.Available volume migration between two pools from the same Ceph cluster.

Out of the scope of the spec:

  2.Available volume migration between Ceph and other vendor driver.

  3.In-use(attached) volume migration using Cinder generic migration.

Proposed change
===============

Solution A: use rbd.Image.copy(dest_ioctx, dest_name) function to migrate
volume from one pool to another pool.

To offload migration operation from cinder-volume host to ceph cluster,
we need to do the following changes in migrate_volume routine in RBD driver.

* Check whether source volume backend and destination volume backend are
  in the same ceph storage cluster or not.

* If not, return (False, None).

* If yes, use rbd.Image.copy(dest_ioctx, dest_name) function to copy volume
  from one pool to another pool.

* Delete the old volume.

Alternatives
------------

Solution B: use ceph's functions of clone image and flatten clone image to
migrate volume from one pool to another pool.

Solution B contains the following steps:
* Create source volume snapshot snap_a and protect the snapshot snap_a.
* Clone a child image image_a of snap_a to the destination pool.
* Flatten the child image image_a, thus snap_a has not been depended on.
* Unprotect the snapshot snap_a and delete it.

Using a volume which's capacity and data size is 12GB to show the
time-consuming comparison between solution A and solution B.

* [Solution-A]Copy volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52
  from "volumes1" pool to "volumes2" pool.

    root@2C5_19_CG1# time rbd cp
    volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52 volumes2/test1

    Image copy: 100% complete...done.

    real    2m3.513s
    user    0m9.983s
    sys     0m25.213s

* [Solution-B-step-1]Create a snapshot of volume
  777617f2-e286-44b8-baff-d1e8b792cc52 and protect it.

    root@2C5_19_CG1# time rbd snap create --pool volumes1 --image
    volume-777617f2-e286-44b8-baff-d1e8b792cc52 --snap snap_test

    real    0m0.465s
    user    0m0.050s
    sys     0m0.016s

    root@2C5_19_CG1# time rbd snap protect
    volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test

    real    0m0.128s
    user    0m0.057s
    sys     0m0.006s

* [Solution-B-step-2]Do clone operation on
  volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test.

    root@2C5_19_CG1# time rbd clone
    volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test
    volumes2/snap_test_clone

    real    0m0.336s
    user    0m0.058s
    sys     0m0.012s

* [Solution-B-step-3]Flatten the clone image volumes2/snap_test_clone.

    root@2C5_19_CG1# time rbd flatten volumes2/snap_test_clone
    Image flatten: 100% complete...done.

    real    1m58.469s
    user    0m4.513s
    sys     0m17.181s

* [Solution-B-step-4]Unprotect the snap
  volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test and
  delete it.

    root@2C5_19_CG1# time rbd snap unprotect
    volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test

    real    0m0.150s
    user    0m0.058s
    sys     0m0.013s

    root@2C5_19_CG1# time rbd snap rm
    volumes1/volume-777617f2-e286-44b8-baff-d1e8b792cc52@snap_test

    real    0m0.418s
    user    0m0.054s
    sys     0m0.011s

By the above test results of solution A and solution B, solution A needs
(real:2m3.513s, user:0m9.983s, sys:0m25.213s) to finish the volume copy
operation and solution B needs (real:1m59.966s, user:0m4.790s, sys:0m17.239s)
to do that. The time-consuming for the both two solutions are not much
difference, but solution A is more simpler than solution B. So we intend to
use solution A to offload volume migration from cinder-volume host to ceph
cluster.

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

The performance of volume migration between two Ceph storage pools
in the same Ceph cluster will be improved greatly.

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
  chen-xueying1<chen.xueying1@zte.com.cn>

Other contributors:
  ji-xuepeng<ji.xuepeng@zte.com.cn>

Work Items
----------

* Add location info of back-end:

  Add location_info in state of Ceph volume service, it should include the
  Ceph cluster name(or id) and storage pool name.

* Implement volume migration:

  1.Check whether the requirements of volume migration are met. If source
  back-end and destination back-end are in the same Ceph cluster and volume
  status is not 'in-use' state, the volume can be migrated.

  2.Copy volume from one pool to another pool and keep it's original image
  name.

  3.Delete the old volume.

Dependencies
============
None

Testing
=======

Unit tests will be added. Volume migration test case will be added.

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change" and ensure that volume migration feature
works well while introducing the feature of RBD volume migration.

Documentation Impact
====================

None

References
==========

None

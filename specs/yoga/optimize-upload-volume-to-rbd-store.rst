..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Optimze upload volume to image for RBD backend
==============================================

https://blueprints.launchpad.net/cinder/+spec/optimize-upload-volume-to-rbd-store

This spec proposes to optimize the upload volume to image operation from cinder
rbd backend to glance rbd store by using COW cloning.

Problem description
-------------------

Currently when doing a upload volume to image operation and the source (cinder)
and destination (glance) backends are rbd, we don't have any optimization
and the generic code path to copy data chunk by chunk is executed.
This can be improved by doing a COW clone as we do it incase of creating volume
from image [1]_.

Use Cases
---------

We want to improve the performance when uploading a volume from cinder rbd backend
to glance rbd store using COW cloning.

Proposed change
---------------

The changes will be needed on both glance and cinder side.

1) Glance

Expose store type and rbd specific store info (like rbd pool name for glance
in this case) via stores info API [2]_.
This can be extended to other stores by providing store specific details with
the stores info API like we do with get pools API [3]_ on cinder side.
Further information regarding its implementation can be provided with a
glance side spec.

2) Cinder

Get the stores info from Glance and find the default store.
If the glance default store is ``rbd`` and cinder is using ``rbd`` backend to
upload the volume to image then we will proceed with the COW clone operation
from the volume pool to the images pool (pool name provided by glance) and
register the image location on glance side.

We will also introduce a new config parameter ``enable_clone_optimization`` to
enable/disable this optimization on cinder side. More information about it
can be found in `Security impact`_ section.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None. This is RBD driver specific feature.

Security impact
---------------

Since this optimization skips the writing of image that happens on the glance side, it will
also skip the checksum and hash value calculated in that scenario.
We will introduce a new config parameter ``enable_clone_optimization`` to enable/disable
this optimization and also mention the security risks that comes with enabling it. By default,
it will be disabled.

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

Uploading volume to image in case of cinder RBD to glance RBD will be
significantly improved.

+------------------------------------+---------------+---------------+---------------+
|             Image size             |      2GB      |     3GB       |      5GB      |
+====================================+===============+===============+===============+
| Time without COW clone             | 1min17Sec     | 1min15Sec     | 2min49Sec     |
+------------------------------------+---------------+---------------+---------------+
| Time with COW clone                | 1.29Sec       | 2.32Sec       | 1.63Sec       |
+------------------------------------+---------------+---------------+---------------+
|                                    | **-98%**      | **-97%**      | **-99%**      |
+------------------------------------+---------------+---------------+---------------+

Other deployer impact
---------------------

The Cinder service user will need to have read and write to the Glance pool.
Note that Read access might already be granted and only write access needs
to be provided.

Developer impact
----------------

None

Implementation
--------------

Assignee(s)
~~~~~~~~~~~

Primary assignee:
  Rajat Dhasmana (whoami-rajat)

Work Items
~~~~~~~~~~

-  Query glance for stores info (After exposing details from glance side)
-  Do a COW clone if the volume to be uploaded is in glance RBD store
   (volume is uploaded in glance default store)

Dependencies
~~~~~~~~~~~~

Glance side changes are required to expose store type and RBD store info
like glance pool name.

Testing
~~~~~~~

*  Unit tests
*  Tempest or manual testing for checking if RBD image dependency blocks
   deletion of source resource

   * Upload volume to image
   * Delete the original volume

   AND

   * Upload volume to image
   * Create volume from image
   * Delete the image

Documentation Impact
~~~~~~~~~~~~~~~~~~~~

None

References
----------

.. [1] https://opendev.org/openstack/cinder/commit/edc11101cbc06bdce95b10cfd00a4849f6c01b33

.. [2] https://docs.openstack.org/api-ref/image/v2/index.html?expanded=list-stores-detail#list-stores

.. [3] https://docs.openstack.org/api-ref/block-storage/v3/?expanded=list-all-back-end-storage-pools-detail#list-all-back-end-storage-pools

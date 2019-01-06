..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Support clone_image from Glance cinder backend
==============================================

https://blueprints.launchpad.net/cinder/+spec/clone-image-in-glance-cinder-backend

When Glance cinder store is enabled, the glance image can store locations of
cinder volume. When a raw image is cinder volume backed, we can efficiently
create a new volume from the image by cloning the backing volume, instead of
downloading the image from Glance. We can also make upload-to-image from a
volume efficient by creating a cloned volume and add its location to an image.

Note that downloading the cinder-backed image from Glance is not currently
supported, but this feature is proposed. [1]_

Problem description
===================

* Creating a new volume from an image, and uploading a volume to an image take
  long time due to image transfer.
* Sharing volume data among tenant is not currently supported in Cinder.
  To share the volume data, we need to upload it to Glance once.

Use Cases
=========

* Share volume templates (e.g. operating system volumes) among tenants via
  cinder volume backed images.
* Each projects can create new instances via the image rapidly. If storage
  array supports thin-provisioning, the disk space could also be efficient.

Proposed change
===============

When the allowed_direct_url_schemes in cinder.conf contain 'cinder',
on creating a volume from an image:

* Check if the image metadata satisfy the following conditions:
    * The image format is raw.
    * The image has locations in the format of 'cinder://<volume-id>'
    * Any of the specified volume is on the same host as a new volume.
    * The image owner and the volume owner is the same (to avoid accessing
      volumes not shared by the image owner).
* If all the conditions are met, clone the specified volume.
  When the requested size is larger than the image, extend the new volume.
* Otherwise, download the image to a new volume.

On uploading a volume to an image:

* If image format is raw and image_upload_use_cinder_backend is enabled in
  cinder.conf, clone the volume and register its location to the image [2]_.

    * If image_upload_use_internal_tenant is set to True, the cloned volume is
      placed in the internal tenant.
* Otherwise, upload the volume data to Glance.

Alternatives
------------

Generic image cache has the same effect in some situations. However, the
proposed feature and generic image cache can co-exist to cover the different
situations.

 * Image cache does not speed up uploading, but this feature make it efficient.
 * image cache can handle non-raw format images (it stores the converted image
   in image volumes), but this proposed feature doesn't handle it so far.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

When the cloned volume is stored in the internal tenant, it could potentially
be a risk if someone was able to get a hold of its credentials or access the
image-volumes. Care will have to be taken to ensure it isn't accessible by
normal users.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

This improves the performance of uploading and download of the image,
especially storage array can provide efficient volume cloning.

Other deployer impact
---------------------

Some configuration options need to be set to enable this feature.

* glance_api_version = 2
* allowed_direct_url_schemes must contain 'cinder'
* image_upload_use_cinder_backend = True (new)
* image_upload_use_internal_tenant = True / False (new)

In Glance, cinder backend must be enabled and show_multiple_locations must be
set to True.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tsekiyama

Other contributors:
  None

Work Items
----------

* Efficient download from cinder volume backed image
* Efficient upload to image

Dependencies
============

None

To enabling the other components to download the cinder volume backed images,
Glance should have the changes [1]_.

Testing
=======

* Unit tests for download from cinder backed image
* Unit tests for upload to image

Documentation Impact
====================

Documentation about how to enable this feature should be added.


References
==========

.. [1] Proposal for Glance to support download from/upload to Cinder backend

* glance-specs: https://review.openstack.org/183363
* glance patch(adding rootwrap): https://review.openstack.org/
* glance_store patch: https://review.openstack.org/166414

.. [2] Proposed change for Cinder

* https://review.openstack.org/#/c/201754/

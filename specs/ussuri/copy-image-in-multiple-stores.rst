..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Copy volume to image in multiple stores of glance
=================================================

https://blueprints.launchpad.net/cinder/+spec/copy-image-in-multiple-stores

This specs propose to send base image reference to glance on volume
upload to image API.

Problem description
===================

In Train, Glance has added the ability to configure multiple stores
[1]_. This way an operator can configure more than one of similar or
different kind of stores and use one as a default store. If a store
is not specified at the time of uploading an image then the image
will be stored in default store.

In Train, I have proposed a spec [2]_ where cinder can specify the glance
store via the volume-type. The store specified in volume-type will be
passed to glance using `x-image-meta-store` header. Even operator
specifies the glance store identifier in volume-type it will only allow
volume image to be uploaded to single specified store and does not allow
for co-location of the image. Also, it depends on the operator configuring
the volume-type extra-specs to define a preferred Glance store; if the
operator doesn't take this step, all uploads go to the default Glance store.

Use Cases
=========

1. At the moment, if cloud provider decided to upgrade their cloud to Train
   release to use the ability of glance of configuring multiple store and the
   base image from which volume is created resides in multiple stores
   of glance, then if the operator has not defined a glance store in the
   volume-type extra specs, the image created from volume will always
   be uploaded to default store and there is no way to replicate/copy these
   image bits to all the stores where base image is uploaded. As a result,
   operators today need to perform a number of manual steps in order
   to replicate/copy image bits on glance stores despite using the
   'enabled_backends' configuration option. For this purpose, he/she need
   to manually copy the newly created image from volume in all stores and
   register these others locations URL with the image using the glance API.

Proposed change
===============

When a volume is created from an image, Cinder currently stores the
image_uuid in the volume_image_metadata. When Cinder upload volume to
image is requested cinder should pass the 'image_uuid' as a header
'x-openstack-base-image-ref' to glance, so that glance can identify the
store(s) in which the original image is located. Cinder is also storing
this glance store information in volume types [2]_, where the store
identifier associated with volume-type will be passed as a header
'x-image-meta-store' to glance to upload the image created from volume to
that specific store.

If a volume is created from an image and Cinder passes both the
'x-image-meta-store' and 'x-openstack-base-image-ref' headers, Glance will
ignore the 'x-openstack-base-image-ref' header and will use the store_id
specified in the 'x-image-meta-store' header.

If volume is not bootable and store information is available in
volume type then glance will use store associated with volume type to upload
the image created from volume.

If volume is not bootable and store information is not available in
volume type then glance will use 'default store' to upload the image
created from volume.

Alternatives
------------

Continue with passing store information using volume types [2]_.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

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

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  whoami-rajat

Other contributors:
  abhishek-kekane

Work Items
----------

* Pass image uuid stored in 'volume_image_metadata' as header to glance
* Add related tests


Dependencies
============

* Depends on implementation of 'Support Glance multiple stores' [2]_.


Testing
=======

* Add related unittest
* Add related functional test
* Add tempest tests


Documentation Impact
====================

Operators documentation should be updated according to spec implementation.


References
==========

.. [1] https://docs.openstack.org/glance/rocky/admin/multistores.html
.. [2] http://specs.openstack.org/openstack/cinder-specs/specs/train/support-glance-multiple-backend.html

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Standardize Image Encryption and Decryption
===========================================

OpenStack already has the ability to create encrypted volumes and ephemeral
storage to ensure the confidentiality of block data. Even though it is also
already possible to store encrypted images, there is only one service (Cinder)
that utilizes this option, but it is only indirectly usable by Nova (a user
must create a volume from the image first), and thus users don't have an
intuitive way to create and upload encrypted images. In addition, all metadata
needed to detect and use encrypted images is either not present or specifically
scoped for Cinder right now. In conclusion, support for encrypted images does
exist to some extent but only in a non-explicit and non-standardized way. To
establish a consistent approach to image encryption for all OpenStack services
as well as users, several adjustments need to be implemented in Glance, Cinder
and OSC.


Problem description
===================

An image, when uploaded to Glance or being created through Nova from an
existing server (VM), may contain sensitive information. The already provided
signature functionality only protects images against alteration. Images may be
stored on several hosts over long periods of time. First and foremost this
includes the image storage hosts of Glance itself. Furthermore it might also
involve caches on systems like compute hosts. In conclusion they are exposed to
a multitude of potential scenarios involving different hosts with different
access patterns and attack surfaces. The OpenStack components involved in those
scenarios do not protect the confidentiality of image data.

Using encrypted storage backends for volume and compute hosts in conjunction
with direct data transfer from/to encrypted images can enable workflows that
never expose an image's data on a host's filesystem. Storage of encryption keys
on a dedicated key manager host ensures isolation and access control for the
keys as well.

As stated in the introduction above, some disk image encryption implementations
for ephemeral disks in Nova and volumes in Cinder already touch on this topic
but not always in a standardized and interoperable way. For example, the way of
handling image metadata and encryption keys can differ. Furthermore, users
are not easily able to make use of these implementations when supplying their
own images in a way that encryption can work the same across services.

That’s why we propose the introduction of a streamlined encrypted image format
along with well-defined metadata specifications which will be supported across
OpenStack services for the existing encryption implementations and increase
interoperability as well as usability for users.

Use Cases
---------

| 1. A user wants to create a new volume based on an encrypted image. The
 corresponding volume host has to be enabled to detect, that the image is
 encrypted. Additionally encrypted images should always result in encrypted
 volumes to avoid decryption.
|
|   1.1 If an encrypted image is the base for a new volume the used volume type
 should always have an encryption type. If the given volume type or default
 volume type does not have an encryption type the operation should result
 in an ERROR.
|
|   1.2 There are two possible types of encrypted images: qcow2 and raw images.
 Cinder can already handle encrypted raw images. Encrypted qcow2 images
 may need to be flattend to raw before transfering them into volumes.
|
|   1.3 Encrypted images, that were created from encrypted volumes, may be
 compressed depending on Cinder's allow_compression_on_image_upload
 option. This also needs to be handled when creating an encrypted volume
 from such an image.

| 2. Whenever an encrypted image is converted to an encrypted volume the secret
 should be copied to give Cinder full control over the life-cycle of the
 secret.
|
|   2.1. The secret can be a key or a passphrase. The secret type
 classification in the Key-Manager will determine the key handling
 ("symmetric" vs "passphrase").  Currently, Cinder is only able to handle
 "symmetric". Support for "passphrase" (like used in Nova) has to be added.

| 3. A user wants to create an image from a volume with an encrypted volume
 type.  The target image will reuse the LUKS encryption and key. This use case
 is already implemented as part of the default behavior of Cinder.
|
|   3.1. Creating an encrypted image from an unencrypted volume will not be
 part of this spec, but may be implemented later on.
|
|   3.2. Creating an unencrypted image from an encrypted volume is not possible
 right now. The volume encryption is transparently also used for the image.
 This behavior will stay in place to optimize resource usage and avoid costly
 conversion of the whole volume data on volume hosts.

Proposed change
===============

In Glance we propose the following additional metadata properties that should be
carried by encrypted images:

* 'os_encrypt_format' - the main mechanism used, e.g. 'LUKS'
* 'os_encrypt_cipher' - the cipher algorithm, e.g. 'AES256'
* 'os_encrypt_key_id' - reference to secret in the key manager
* 'os_encrypt_key_deletion_policy' - on image deletion indicates whether the key
  should be deleted too
* 'os_decrypt_container_format' - format change, e.g. from 'compressed' to
  'bare'
* 'os_decrypt_size' - size after payload decryption

We propose to align the encryption with Nova and Cinder and use LUKS, which
will be allowed in combination with qcow and raw images. We use this two
versions for the following reasons:

1. Nova can directly use qcow-LUKS encrypted when creating a server. This is
   the standard procedure of Nova.

2. Cinder allows the creation of Images from encrypted volumes. These will
   always result in LUKS-encrypted raw images. Those images can be converted
   directly to volumes again.

In the latter case it is already possible to upload such an encrpyted image to
another OpenStack infrastructure, upload the key as well and set the
corresponding metadata. After doing so the image can be used in the second
infrastructure to create an encrypted volume.

We want to align the existing implementations between Nova and Cinder by
standardizing the used metadata parameters and adding interoperability where
applicable. This would in the case of Cinder mainly be a rename:
- 'cinder_encryption_key_deletion_policy' to 'os_encrypt_key_deletion_policy'
- 'cinder_encryption_key_id' to 'os_encrypt_key_id'

In the Volume creation there will be a check added, to look for encrypted
images proposed as a volume source. If an image is encrypted another check is
added to determine, whether the volume type to create the volume has an
encryption type. If that is not the case the volume create will be aborted in
the API still. Otherwise there will be an unusable volume created.

The flattening of a qcow2 image should be handled when uploading the image to
the volume. The volume size should be resulting from the 'os_decrypt_size'
parameter. If compression is enabled through Cinder's
allow_compression_on_image_upload option Cinders implementation to handle this
should be re-used.

The key management for creating an encrypted volume from an encrypted image
must include the copying of the secret in Barbican. On this way Cinder always
has the full control over the life-cycle of the secret, because this way is
similar to the original creation of an encrypted volume.

In all places that use decryption within Cinder, there need to be an
additional check for the type of the secret. And a different handling, if the
secret is a "passphrase", because the way Cinder treats keys to create a
passphrase for the LUKS header of a volume is quite unique and differs from
Nova's handling of images, that have passphrases only.

The creation of an image from an volume just need to be adjusted to use the new
parameters.


Alternatives
------------

We also evaluated an image encryption implementation based on GPG. The downside
with such an implementation is, that everytime such an image is used to create
a server or a volume the image has to be decrypted and maybe re-encrypted for
another encryption format as both Nova and Cinder use LUKS as an encryption
mechanism. This would not only have impact on the performance of the operation
but it also would need free space for the encrypted image file, the decrypted
parts and the encrypted volume or server that is created.


Data model impact
-----------------

None


REST API impact
---------------

When creating a volume from an encrypted image there might occure a new ERROR
that is triggered, when an image is encrypted but no encrypted volume type is
given.


Security impact
---------------

There are impacts on the security of OpenStack:

* confidentiality of data in images will be addressed in this spec

* image encryption is introduced formally, thus cryptographic algorithms will
  be used in all involved components (Nova, Cinder, OSC)


Active/Active HA impact
-----------------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------

* Users should be able to use encrypted images to create volumes in a
  consistant way


Performance Impact
------------------

The proposed checks for the Cinder API may have minimal impact on performance.

When creating a volume or server from an encrypted image the only operation
that may be triggerd is the conversion between qcow-LUKS and raw LUKS blocks.

Thus, any performance impact is only applicable to the newly introduced
encrypted image type where the processing of the image will have increased
computational costs and longer processing times than regular images. Impact
will vary depending on the individual host performance and supported CPU
extensions for cipher algorithms.


Other deployer impact
---------------------

* For interoperability between the OpenStack services only the presence of a
  key manager should decide, whether encryption can be used or not.

* A key manager - like Barbican - is required, if encrypted images are to be
  used.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee: Markus Hentsch (IRC: mhen)

Other contributors: Josephine Seifert (IRC: Luzi)

Work Items
----------

* Add checks in the create volume API

* Add copying the secret and registering as a consumer in Barbican

* Add flattening of qcow2 to raw encrypted images

* In the image create from volume: change the
  'cinder_encryption_key_deletion_policy' to 'os_encrypt_key_deletion_policy'
  and 'cinder_encryption_key_id' to 'os_encrypt_key_id'


Dependencies
============

* Presence of the image encryption parameters in Glance has to be implemented


Testing
=======

Tempest tests would require access to encrypted images for testing. This means
that Tempest either needs to be provided with an image file that is already
encrypted and its corresponding key or needs to be able to encrypt images
itself. This point is still open for discussion.


Documentation Impact
====================

It should be documented for deployers, how to enable this feature in the
OpenStack configuration.


References
==========

[1] Barbican Secret Consumer Spec:
https://review.opendev.org/#/c/662013/

[2] Glance Image Encryption Spec:
https://review.opendev.org/c/openstack/glance-specs/+/915726


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Dalmatian
     - Introduced

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Standardize Image Encryption and Decryption
===========================================

OpenStack already supports encrypted volumes and ephemeral storage to ensure
the confidentiality of block data. Even though it is already possible to store
encrypted images, there is only one service (Cinder) that utilizes this option,
but it is only indirectly usable by Nova (a user must create a volume from the
image first), and thus users do not have an intuitive way to create and upload
encrypted images. In addition, all metadata needed to detect and use encrypted
images is either not present or specifically scoped for Cinder right now. In
conclusion, support for encrypted images does exist to some extent but only in
a non-explicit and non-standardized way. To establish a consistent approach to
image encryption for all OpenStack services as well as users, several
adjustments need to be implemented in Glance, Cinder, and OSC.


Problem description
===================

An image, when uploaded to Glance or created through Nova from an existing
server (VM), may contain sensitive information. The already provided signature
functionality only protects images against alteration. Images may be stored on
several hosts over long periods of time. First and foremost this includes the
image storage hosts of Glance itself. Furthermore it might also involve caches
on systems like compute hosts. In conclusion, the images are exposed to a
multitude of potential scenarios involving different hosts with different
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
interoperability as well as usability for users. Additionally we require, that
encrypted images will always result in encrypted volumes to avoid decryption.

Use Cases
---------

1. A user wants to create a new volume based on an encrypted image. The
corresponding volume host has to be enabled to detect that the image is
encrypted.

   a. If an encrypted image is the base for a new volume the used volume type
      should always have an encryption type. If the given volume type or
      default volume type does not have an encryption type the operation
      should result in an ERROR.

   b. There are two possible types of encrypted images: qcow2 images with
      encryption (qcow2+LUKS) and raw LUKS images. Cinder can already handle
      raw LUKS-encrypted images. Encrypted qcow2+LUKS images may need to be
      converted to raw LUKS before transforming them into volumes.

2. Whenever an encrypted image is converted to an encrypted volume the secret
should be copied to give Cinder full control over the lifecycle of the secret.

   a. There is only one secret per image and it can be either a key or a
      passphrase. The secret type classification in the Key Manager will
      determine the key handling ("symmetric" vs "passphrase"). In case of
      "passphrase", the secret is passed directly as the passphrase to the
      encryption layer. In any othercase, it is interpreted as binary and
      converted to a suitable string representation before being passed to the
      encryption. Currently, Cinder is only able to handle "symmetric" keys,
      i.e., the latter binary case. Support for "passphrase" (like used in
      Nova) has to be added.

3. A user wants to create an image from a volume with an encrypted volume type
The target image will reuse the LUKS encryption and key. This use case is
already implemented as part of the default behavior of Cinder.

   a. Creating an encrypted image from an unencrypted volume will not be part
      of this spec, but may be implemented later on.

   b. Creating an unencrypted image from an encrypted volume is not possible
      right now. The volume encryption is transparently also used for the
      image. This behavior will stay in place to optimize resource usage and
      avoid costly conversion of the whole volume data on volume hosts.



Proposed change
===============

In Glance we propose the following additional metadata properties that should be
carried by encrypted images:

* 'os_encrypt_format' - the specific mechanism used, e.g. 'LUKSv1'
* 'os_encrypt_key_id' - reference to key in the key manager
* 'os_encrypt_key_deletion_policy' - on image deletion indicates whether the
  key should be deleted too
* 'os_decrypt_size' - size after payload decryption

We propose to align the encryption with Nova and Cinder and use LUKS, which
will be allowed in combination with qcow2 and raw images. We use these two
variations for the following reasons:

1. Nova might be able to directly use qcow2+LUKS encrypted images when creating
   a server. The qcow2 format is already familiar to Nova and used by it to
   directly boot instances from. Currently, support for encrypted ephemeral
   storage is not yet implemented in Nova, so qcow2+LUKS cannot be used by it
   yet but since the LUKS encryption is a native feature of qcow2, this
   will provide a good starting point for a lightweight future extension.

2. Cinder allows the creation of images from encrypted volumes. These will
   always result in LUKS-encrypted raw images. Those images can be converted
   directly to volumes again. This behavior is already implemented and requires
   no format conversion as the LUKS encryption is native to Cinder.
   The intention is to keep this functionality and make the format usable
   outside of Cinder and provide interoperability for it.
   To better identify such an image, we propose the new 'container_format'
   'luks' to be set for these images.

In the latter case it is already possible to upload such an encrypted image to
another OpenStack deployment, upload the key as well and set the
corresponding metadata. After doing so the image can be used in the second
deployment to create an encrypted volume.

We want to align the existing implementations between Nova and Cinder by
standardizing the used metadata parameters and adding interoperability where
applicable. This would in the case of Cinder mainly be a change in the naming
of the following image properties:

- 'cinder_encryption_key_deletion_policy' to 'os_encrypt_key_deletion_policy'
- 'cinder_encryption_key_id' to 'os_encrypt_key_id'

A check in the volume creation flow will be added to look for encrypted
images proposed as a volume source. If an image is encrypted, another check is
added to determine, whether the volume type used to create the volume has an
encryption type. If that is not the case the volume creation will be aborted
early in the API. Since the encrypted data is always directly transferred over,
the volume would end up as unusable otherwise.

The conversion of a qcow2+LUKS image should be handled when downloading the
image to a volume. The required volume size should be determined based on the
'os_decrypt_size' property of the image.

The key management for creating an encrypted volume from an encrypted image
must include the cloning of the secret in Barbican. This way Cinder always
has the full control over the lifecycle of the secret and is similar to the
original creation of an encrypted volume. As secrets used for encrypted images
may be user-defined, Cinder cannot rely on the existence of the original secret
over time.

In all places that implement image decryption within Cinder, an additional
check for the type of the secret needs to be added. Different key handling
needs to be triggered if the secret is a "passphrase", because the way Cinder
currently treats keys to create a passphrase for the LUKS header of a volume
in way that always attempts to convert binary keys to an ASCII encoding first
and differs from Nova's handling of images, that directly passes passphrases
only.

The creation of an image from a volume just needs to be adjusted to use the new
properties.


Alternatives
------------

We also evaluated an image encryption implementation based on GPG. The downside
with such an implementation is, that every time such an image is used to create
a server or a volume the image has to be decrypted and maybe re-encrypted for
another encryption format as both Nova and Cinder use LUKS as an encryption
mechanism. This would not only have an impact on the performance of the
operation but it also would need free space for the encrypted image file, the
decrypted parts and the encrypted volume or server that is created.


Data model impact
-----------------

None


REST API impact
---------------

When creating a volume from an encrypted image there might occur a new ERROR
that is triggered when an image is encrypted but no encrypted volume type is
given.


Security impact
---------------

There are impacts on the security of OpenStack:

* confidentiality of data in images will be addressed in this spec

* image encryption is introduced formally

* cryptographic algorithms will be used in all involved components
  (Nova, Cinder, OSC) that work with encrypted images


Active/Active HA impact
-----------------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------

* Users should be able to use encrypted images to create volumes in a
  consistent way


Performance Impact
------------------

The proposed checks for the Cinder API may have minimal impact on performance.

When creating a volume or server from an encrypted image the only operation
that may be triggered is the conversion between qcow2+LUKS and raw LUKS blocks.

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

* Add cloning the secret for image encryption keys

* Add dedicated handling of "passphrase" type secrets via os-brick

* Add conversion of qcow2+LUKS images to raw LUKS encrypted block data

* In the image creation from volume: change the
  'cinder_encryption_key_deletion_policy' to 'os_encrypt_key_deletion_policy'
  and 'cinder_encryption_key_id' to 'os_encrypt_key_id'

* Add setting the 'os_encrypt_format' property when creating images from
  encrypted volumes


Dependencies
============

* Handling of the new image encryption parameters and secret consumer usage
  in Glance has to be implemented

* os-brick implements a shared utility function for converting encryption keys
  of different types, including 'passphrase' and 'symmetric'


Testing
=======

Tempest tests will be added to the barbican-tempest-plugin in addition to its
existing scenario tests revolving around usage of secrets. These scenario
tests will create encrypted images in various permutations, including the
different image formats (qcow2+LUKS, raw LUKS) as well as the different secret
types influencing the key conversion. Each of the created images will be used
to create a volume from which a Nova instance will be booted and
health-checked.

Another scenario will be added that specifically tests the creation of
encrypted images based on an encrypted volume by Cinder and the subsequent
use for a new encrypted volume to verify the correct production and
consumption of encrypted images as implemented by Cinder itself. This will act
as a regression test to make sure that the existing functionality of Cinder
being able to backup and restore encrypted volumes to/from images while
keeping the encryption intact through the whole chain stays functional.


Documentation Impact
====================

It should be documented for deployers, how to enable this feature in the
OpenStack configuration.


References
==========

[1] Barbican Secret Consumer Spec:
https://review.opendev.org/#/c/662013/

[2] Glance Image Encryption Spec:
https://review.opendev.org/c/openstack/glance-specs/+/964755

[3] Extended key to passphrase conversion outsourced to os-brick:
https://review.opendev.org/c/openstack/os-brick/+/926293

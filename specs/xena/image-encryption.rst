..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Image Encryption and Decryption
==========================================

OpenStack already has the ability to create encrypted volumes and ephemeral
storage to ensure the confidentiality of block data. In contrast to that,
images are currently handled without protection towards confidentiality,
only providing the possibility to ensure integrity using image signatures.
For further protection of user data - e.g. when a user uploads an image
containing private data or confidential information - the image data should
not be accessible for unauthorized entities. For this purpose, an encrypted
image format is to be introduced in OpenStack.

To enable this, several adjustments to support image encryption/decryption in
various projects, e.g. Nova, Glance, Cinder and OSC, need to be implemented.

The proposal documented in this spec describes the Cinder-related part of
the overall cross-project concept for image encryption.


Problem description
===================

An image, when uploaded to Glance or being created through Nova from an
existing server (VM), may contain sensitive information. The already
provided signature functionality only protects images against alteration.
Images may be stored on several hosts over long periods of time. First and
foremost this includes the image storage hosts of Glance itself. Furthermore
it might also involve other systems such as volume hosts. As a result they
are exposed to a multitude of potential scenarios involving different hosts
with different access patterns and attack surfaces. The OpenStack components
involved in those scenarios – including Cinder - do not protect the
confidentiality of image data.

Using encrypted storage backends for volume and compute hosts in conjunction
with direct data transfer from/to encrypted images can enable workflows that
never expose an image's data on a host's filesystem. Storage of encryption
keys on a dedicated key manager host ensures isolation and access control for
the keys as well. With such a set of configuration recommendations for
security focused environments, we are able to reach a good foundation. Future
enhancements can build upon that and extend the provided security enhancements
to a broader set of variants.

That’s why we propose the introduction of an encrypted image format.


Use Cases
---------

1. A user wants to create a new volume based on an encrypted image. The
corresponding volume host has to be able to decrypt the image using the
symmetric key stored in the key manager and transform it into the requested
volume resource.

2.1. A user wants to create an encrypted image from a volume. The volume
does not use an encrypted volume type. The target image is to be encrypted
using the proposed image encryption mechanism and a specified key.

2.2. A user wants to create an encrypted image from a volume. The volume uses
an encrypted volume type (LUKS-based). The target image is not to be
re-encrypted but will instead reuse the LUKS encryption and key. This use
case is already implemented as part of the default behavior of Cinder.

2.3. A user wants to create an encrypted image from a volume. The volume uses
an encrypted volume type (LUKS-based). The target image is to be re-encrypted
using the proposed image encryption mechanism and a specified key. This
involves opening a dmcrypt mapping for the LUKS encryption and streaming the
plain data of the volume directly into the target image file while encrypting
it using the specified image encryption mechanism (e.g. GPG).


Proposed change
===============

In Glance we propose to add a new container_format called 'encrypted'.
Furthermore, we propose the following additional metadata properties carried by
images of this format:

* 'os_encrypt_format' - the main mechanism used, e.g. 'GPG'
* 'os_encrypt_type'   - encryption type, e.g. 'symmetric'
* 'os_encrypt_cipher' - the cipher algorithm, e.g. 'AES256'
* 'os_encrypt_key_id' - reference to key in the key manager
* 'os_decrypt_container_format' - format after payload decryption
* 'os_decrypt_size' - size after payload decryption

All these metadata properties are only used for the decryption of an image and
won't be needed in the lifecycle of a volume anymore. When creating an image
from a volume these metadata properties are newly generated and saved in the
Glance metadata of the new image.

For Cinder we want to add support for encrypted image decryption when a
volume is created from it. Cinder should select a suitable decryption
mechanism according to the new image container_format and metadata. The
symmetric key will be retrieved from the key manager (e.g. Barbican)
according to the key id stored in the metadata. Additionally, Cinder should
be able to encrypt volume data in a similar fashion if the user requests an
encrypted image to be created from a volume according to the use cases 2.1
and 2.3 illustrated above.

The implementations in Cinder should make use of a centralized encryption
implementation provided by a global library, shared between all involved
OpenStack components to prevent individual implementations of the encryption
mechanism.

For general image encryption, we propose to use an implementation of
symmetric AES 256 provided by GnuPG as a basic mechanism supported by this
draft. It is a well established implementation of PGP and supports
streamable encryption/decryption processes, which is important as we require
the streamability of the encryption/decryption mechanism for two reasons:

1. Loading entire images into the memory of volume hosts is unacceptable.

2. We propose direct decryption-streaming into the target volume to prevent
   the creation of temporary unencrypted files.

There is already one existing case in Cinder’s current implementation where
encrypted images are created (use case 2.2). This is when an image is
created directly from an encrypted volume. Since the encrypted block data is
simply copied into the image, the encryption (usually LUKS) is automatically
inherited - as is the encryption key, which is simply cloned in Barbican.
We will not change this behavior as a part of this spec. Our changes will
only apply, when the user actively wants to create an encrypted image from
any volume.

The key management is handled differently than with encrypted volumes or
encrypted ephemeral storage. The reason for this is, that the encryption and
decryption of an image will never happen in Glance but in all other services,
which consume images. Therefore the service which needs to create a key for
a newly created encrypted image may not be the same service which then has to
delete the key (in most cases Glance). To delete a key, which has not been
created by the same entity, is bad behavior. To avoid this, we choose to let
the user create and delete the key. To not accidently delete a key, which is
used to encrypt an image, we will let Glance register as a consumer of that
key (secret in Barbican [1]) when the corresponding encrypted image is
uploaded and unregister as a consumer when the image is deleted in Glance.

The methods for encryption and decryption of files - in this case images -
will be written in a driver like manner in os-brick so the image encryption
can be extended with another encryption format easily. The encryption driver
should focus a specific encryption format and implement exactly one encrypt
and one decrypt method, both based on a cipher implementation of GPG aes.
This driver may be simple wrappers around an existing implementation. An
abstract base class should be defined and be used for the implementation of
GPG encryption (and might be used for other implementations in the future).


Alternatives
------------

Regarding the image encryption, we also explored the possibility of using
more elaborate and dynamic approaches like PKCS#7 (CMS) but ultimately
failed to find a free open-source implementation (e.g. OpenSSL) that
supports streamable decryption of CMS-wrapped encrypted data. More precisely,
no implementation we tested was able to decrypt a symmetrically encrypted,
CMS-wrapped container without trying to completely load it into memory or
suffering from other limitations regarding big files.

We also evaluated an image encryption implementation based on LUKS which is
already used in Cinder and Nova as an encryption mechanism for volumes and
ephemeral disks respectively. However, we were unable to find a suitable
solution to directly handle file-based LUKS encryption in user space. Firstly,
the handling of LUKS devices (even when file-based) via cryptsetup always
requires the dm-crypt kernel module and corresponding root privileges.
Secondly, in contrast to native LUKS used by LibVirt, the LUKS handling
available via cryptsetup creates temporary device mapper endpoints where data
is read from or written to. There is no direct reading/writing from/to an
encrypted LUKS file and LUKS opening/closing needs to be handled accordingly.
Lastly, LUKS is a format primarily designed for disk encryption. Although it
may be used for files as well (by formatting files as LUKS devices), the
handling is rather inconvenient; for example, the size of the LUKS container
file needs to be calculated and allocated beforehand since it acts like a disk
with a fixed size.


Data model impact
-----------------

None


REST API impact
---------------

For creating encrypted images from volumes, additional properties in the
request body of “os-volume_upload_image” will need to be introduced to
specify the desired encryption format and key id.


Security impact
---------------

There are impacts on the security of OpenStack:

* confidentiality of data in images will be addressed in this spec

* image encryption is introduced, thus additional cryptographic algorithms
  will be used in Cinder to implement this functionality


Notifications impact
--------------------

None


Other end user impact
---------------------

* Users should be able to optionally, but knowingly create a differently
  encrypted image from a volume.

* If an administrator has configured Glance to reject unencrypted images, such
  images will not be accepted when attempted to be uploaded to Glance.


Performance Impact
------------------

The proposed encryption/decryption mechanisms in Cinder will only be utilized
on-demand and skipped entirely for image container types that aren’t
encrypted.

Thus, any performance impact is only applicable to the newly introduced
encrypted image type where the processing (or creation) of such image will
have increased computational costs and longer processing times than for
regular images. Impact will vary depending on the individual host performance
and supported CPU extensions for cipher algorithms.

For Ceph-based setups, the usual cloning performance benefit provided by the
shared storage between Glance and Cinder is lost for encrypted images, due to
the need of converting the encrypted image data format to the volume format.


Other deployer impact
---------------------

* A key manager - like Barbican - is required.

* The key manager needs to be accessible from volume hosts, requiring
  appropriate cinder.conf adjustments.


Developer impact
----------------

* To use the encoding and decoding of images in os-brick, we need to
  execute priviledged functions. We decided to use privsep for this as in
  nova.


Upgrade impact
--------------

none


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Markus Hentsch (IRC: mhen)

Other contributors:
  Josephine Seifert (IRC: Luzi)


Work Items
----------

* Add a decryption implementation in Cinder for creating volumes from GPG
  encrypted images

* Add a dedicated encryption workflow and implementation in Cinder for
  creating encrypted images from volumes using the proposed image encryption
  format (GPG)

* Add encryption and decryption methods for the GPG format in os-brick


Dependencies
============

* GPG is required to be installed on all systems that are required to
  perform encryption/decryption operations in order to support the proposed
  base encryption mechanism.

* This spec requires the implementation of an encrypted container_format and
  corresponding metadata property support in Glance

* This spec requires the implementation of appropriate encryption/decryption
  functionality in a global library shared between the components involved
  in image encryption workflows (Nova, Cinder, OSC), like os-brick


Testing
=======

Tempest tests would require access to encrypted images for testing. This
means that Tempest either needs to be provided with an image file that is
already encrypted and its corresponding key or needs to be able to encrypt
images itself. This point is still open for discussion.


Documentation Impact
====================

It should be documented for deployers, how to enable this feature in the
OpenStack configuration. An end user should have a documentation on how to
use encrypted images in Cinder and how to create them respectively.
Furthermore, any resulting limitations such as reduced performance should be
mentioned, especially for Ceph-based setups usually benefiting from shared
storage cloning.


References
==========

[1] Barbican Secret Consumer Spec: https://review.opendev.org/#/c/662013/

Nova-Spec: https://review.openstack.org/#/c/608696/

Glance-Spec: https://review.openstack.org/#/c/609667/


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
   * - Ussuri
     - Postponed

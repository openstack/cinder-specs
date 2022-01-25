..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Support Image Signature Verification
====================================

https://blueprints.launchpad.net/cinder/+spec/cinder-support-image-signing

Cinder currently does not support signature validation of downloaded signed
images. Equipping Cinder with the ability to validate image signatures will
provide end users with stronger assurances of the integrity of the image data
they are using to create volumes. This change will use the same data model for
image metadata as the accompanying functionality in Glance, which will allow
the end user to sign images and verify these image signatures upon
upload [1]_.

Problem description
===================

Previously, OpenStack's protection against unexpected modification of images
was limited to verifying an MD5 checksum. While this may be sufficient for
protecting against accidental modifications, MD5 is a hash function, not an
authentication primitive [2]_, and thus provides no protection against
deliberate, malicious modification of images. An image could potentially be
modified in transit, such as when it is uploaded to Glance or transferred to
Cinder. An image that is modified could include malicious code.

Currently, Glance has support for image signature verification upon upload,
but Cinder does not support the feature to ensure the integrity of the image
data before using it. Providing support for signature verification would allow
Cinder to verify the signature before creating volume from image. This feature
will secure OpenStack against the following attack scenarios:

* Man-in-the-Middle Attack - An adversary with access to the network between
  Cinder and Glance is altering image data as Cinder downloads the data from
  Glance. The adversary is potentially incorporating malware into the image
  and/or altering the image metadata.

* Untrusted Glance - In a hybrid cloud deployment, Glance is hosted on
  machines which are located in a physically insecure location or is hosted by
  a company with limited security infrastructure. Adversaries may be able to
  compromise the integrity of Glance and/or the integrity of images stored by
  Glance through physical access to the host machines or through poor network
  security on the part of the company hosting Glance.

Please note that our threat model considers only threats to the integrity of
images while they are in transit between the end user and Glance, while they
are at rest in Glance and while they are in transit between Glance and Cinder.
This threat model does not include, and this feature therefore does not
address, threats to the integrity, availability, or confidentiality of Cinder.

Use Cases
=========

* A user wants a high degree of assurance that a customized image which they
  have uploaded to Glance has not been accidentally or maliciously modified
  prior to creating a volume from the image.

With this proposed change, Cinder will verify the signature of a signed image
while downloading that image. If the image signature cannot be verified, then
Cinder will not create a volume from the image and instead place the volume
into an error state. The user will begin to use this feature by uploading the
image and the image signature metadata to Glance via the Glance API's
image-create method. The required image signature metadata properties are as
follows:

* img_signature - A string representation of the base 64 encoding of the
  signature of the image data.

* img_signature_hash_method - A string designating the hash method used for
  signing. Currently, the supported values are  SHA-224, SHA-256, SHA-384 and
  SHA-512. MD5 and other cryptographically weak hash methods will not be
  supported for this field. Any image signed with an unsupported hash
  algorithm will not pass validation.

* img_signature_key_type - A string designating the signature scheme used to
  generate the signature. For more detail on which are currently supported,
  please check Glance's documentation [7]_.

* img_signature_certificate_uuid - A string encoding the certificate
  uuid used to retrieve the certificate from the key manager.

The image verification functionality in Glance uses the
cursive.signature_utils module to verify this signature metadata before
storing the image. If the signature is not valid or the metadata is
incomplete, this API method will return a 400 error status and put
the image into a "killed" state. Note that, if the signature metadata
is simply not present, the image will be stored as it would normally.

The user would then create a volume from this image using the Cinder API's
volume create method. If the verify_glance_signatures flag in cinder.conf is
set to 'True', Cinder will call out to Glance for the image's properties,
which include the properties necessary for image signature verification.
Cinder will pass the image data and image properties to the signature
verification module, which will verify the signature. If signature
verification fails, or if the image signature metadata is either incomplete
or absent, creating volume from the image will fail and Cinder will log an
exception. If signature verification succeeds, Cinder will create volume
from the image and log a message indicating that image signature verification
succeeded along with detailed information about the signing certificate.

Proposed change
===============

Since Nova has implemented this feature and all of the verification process
has been moved into ``cursive`` module [4]_, it's more convenient to support
this in Cinder now.

**Verify image signature with certificate**

We propose an initial implementation by incorporating ``cursive`` into
Cinder's control flow for Creating volumes from images, and use new
configuration ``verify_glance_signatures`` to turn this on or off.
Initially it will have two options (default is ``enabled``):

1. ``enabled``: verify when image has complete signature metadata.
2. ``disabled``: verification is turned off.

**NOTE**: We have discussed to add ``required`` option to introduce a
strict mode on verification, but this can't be guaranteed as we can't
do verification when image volume is cloned in backend. Strict mode will
still be considered when we can cover every approach.

Upon downloading an image, Cinder will both check the new configuration
flag and image's metadata. If needs, the module will perform image signature
verification using image properties passed to Cinder by Glance. If this fails,
or if the image signature metadata is incomplete or missing, Cinder will not
create the volume from the image. Instead, Cinder will throw an exception and
log an error. If the signature verification succeeds, Cinder will proceed with
creating the volume. The code sample is below::

 if CONF.glance.verify_glance_signatures != 'disabled':
    verifier = None
    image_meta_dict = self.show(context, image_id,
                                include_locations=False)
    image_meta = objects.ImageMeta.from_dict(image_meta_dict)
    img_signature = image_meta.properties.get('img_signature')
    img_sig_hash_method = image_meta.properties.get(
        'img_signature_hash_method'
    )
    img_sig_cert_uuid = image_meta.properties.get(
        'img_signature_certificate_uuid'
    )
    img_sig_key_type = image_meta.properties.get(
        'img_signature_key_type'
    )
    try:
        verifier = signature_utils.get_verifier(
            context=context,
            img_signature_certificate_uuid=img_sig_cert_uuid,
            img_signature_hash_method=img_sig_hash_method,
            img_signature=img_signature,
            img_signature_key_type=img_sig_key_type,
        )
    except cursive_exception.SignatureVerificationError:
        #Image signature verification failed
    # Collect image data
    try:
        if verifier:
            verifier.verify()
    except cryptography.exceptions.InvalidSignature:
        #Image signature verification failed

**NOTE**: We will try different approaches when
creating volume from images, so we have to mention
this feature will not cover every approach especially
when volume is created at backend.

To be clear, we will verify the image's signature only when
image is downloaded from glance and content is copied to
volume on host. So when image volume is created via
``clone_image`` or ``clone_image_volume`` we will skip this
verification process regardless of configuration option and
provided signature metadata, in order not to confuse end users,
we will add verification flag ``signature_verified`` in volume's
image metadata when creating from image.

**Verify certificate with trusted certificates**

This feature tries to find a way to determine if the certificate
used to generate and verify that signature is a certificate that
is trusted by the user, we could find more detail in Nova spec [5]_.
In short, within that feature end user can also validate the image's
certificate with the given trusted certificates (specified via API
or config option).
Considering the feature is in the process of being added to Nova
now, we will follow this up with another spec when it's merged
into Nova for the purpose of consistency.

Alternatives
------------

An alternative to signing the image's data directly is to support signatures
which are created by signing a hash of the image data. This introduces
unnecessary complexity to the feature by requiring an additonal hashing stage
and an additional metadata option. Due to the Glance community's performance
concerns associated with hashing image data, the developers initially pursued
an implementation which produced the signature by signing an MD5 checksum
which was already computed by Glance. This approach was rejected by the Nova
community due to the security weaknesses of MD5 and the unnecessary complexity
of performing a hashing operation twice and maintaining information about both
hash algorithms.

An alternative to using certificates for signing and signature verification
would be to use a public key. However, this approach presents the significant
weakness that an attacker could generate their own public key in the key
manager, use this to sign a tampered image, and pass the reference to their
public key to Cinder along with their signed image. Alternatively, the use of
certificates provides a means of attributing such attacks to the certificate
owner, and follows common cryptographic standards by placing the root of trust
at the certificate authority.

An alternative to using the verify_glance_signatures configuration flag to
specify that Cinder should perform image signature verification is to use
a "verify_glance_signatures" type-key for a volume type to specify that
individual volume should be created from signed images. The user, when using
the Cinder CLI to create a volume from image, would specify a volume type
which includes a type-key "verify_glance_signatures=True" to indicate that
image signature verification should occur as part of the control flow for
creating the volume. This may be added in a later change, but will not be
included in the initial implementation. If added, the
"verify_glance_signatures" type-key option will work alongside the
configuration option approach. In this case, Cinder would perform image
signature verification if either the configuration flag is set, or if the user
has specified creating a volume of the volume type which includes
"verify_glance_signatures=True" type-key.

Another alternative to using the verify_glance_signatures configuration flag
to specify that Cinder should perform image signature verification is amending
the Cinder create command to accept an additional parameter specifying whether
image signature verification should occur. This may be added in a later
change, but will not be included in the initial implementation. If added, the
additional parameter will work alongside the configuration option approach.
In this case, Cinder would perform image signature verification if either the
configuration flag is set, or if the user has specified creating a volume of
the additional parameter.

We maybe only need to choose one of the above two alternatives.

Data model impact
-----------------

The accompanying work in Glance introduced additional Glance image properties
necessary for image signing. The initial implementation in Cinder will
introduce a configuration flag indicating whether Cinder should perform image
signature verification before booting an image.

REST API impact
---------------

None

Security impact
---------------

Cinder currently lacks a mechanism to validate images prior to creating
volumes from them. The checksum included with an image protects against
accidental modifications but provides little protection against an adversary
with access to Glance or to the communication network between Cinder and
Glance. This feature facilitates the creation of a logical trust boundary
between Cinder and Glance; this trust boundary permits the end user to have
high assurance that Cinder is creating a volume from an image signed by a
trusted user.

Although Cinder will use certificates to perform this task, the certificates
will be stored by a key manager and accessed via Castellan.

Notifications impact
--------------------

This change will involve adding log messages to indicate the success or
failure of signature verification and creation.

A later change will involve notifying the user about failure in case signature
verification fails, this will use async error notification feature [3]_.

Other end user impact
---------------------

If the verification of a signature fails, then Cinder will not create a
volume from the image, and an error message will be logged and recorded.
The user can get the error messages through the log file or CLI command,
and know the reason for the error. In this case, the user will have to
edit the image's metadata through the Glance API, or the Horizon interface;
or reinitiate an upload of the image to Glance with the correct signature
metadata in order to create a volume from the image.

Performance Impact
------------------

This feature will only be used if the verify_glance_signatures configuration
flag is set.

When signature verification occurs there will be latency as a result of
retrieving certificates from the key manager through the Castellan interface.
There will also be CPU overhead associated with hashing the image data and
decrypting a signature using a public key.

Other deployer impact
---------------------

We will recommend you deploy Barbican service [6]_ to store your
certificate information as other projects suggest, although you can integrate
any other secret manager service via Castellan [8]_.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ji-xuepeng
  TommyLike(tommylikehu@gmail.com)

Other contributors:
  None

Work Items
----------

The feature will be implemented in the following stages:

* Add functionality to Cinder which calls the ``cursive`` module when Cinder
  downloads a Glance image and the verify_glance_signatures configuration flag
  is set.

* Generate notification messages at the time of failure for signature
  verification.

Dependencies
============

None

Testing
=======

Unit tests and also, tempest tests will be added into
barbican-tempest-plugin to cover the case of create
volume from signed image.


Documentation Impact
====================

Instructions for how to use this functionality will need to be documented.


References
==========

Cryptography API: https://pypi.org/project/cryptography/0.2.2

.. [1] https://review.openstack.org/#/c/252462/
.. [2] https://en.wikipedia.org/wiki/MD5#Security
.. [3] https://blueprints.launchpad.net/cinder/+spec/summarymessage
.. [4] https://git.openstack.org/cgit/openstack/cursive
.. [5] http://specs.openstack.org/openstack/nova-specs/specs/queens/approved/nova-validate-certificates.html
.. [6] https://docs.openstack.org/barbican/latest/
.. [7] https://docs.openstack.org/glance/pike/user/signature.html
.. [8] https://wiki.openstack.org/wiki/Castellan

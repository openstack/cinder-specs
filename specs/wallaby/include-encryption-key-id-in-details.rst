..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Include encryption_key_id in volume and backup details
======================================================

https://blueprints.launchpad.net/cinder/+spec/include-encryption-key-id-in-details

This spec adds the encryption_key_id field to the volume and backup details
for volumes that are encrypted.


Problem description
===================

The current API supplies volume details that include a simple boolean that
indicates whether a volume is encrypted. However, when a volume is encrypted,
the details do not include the associated Key Manager (e.g. Barbican)
encryption_key_id. This makes it difficult to correlate encryption keys stored
in Barbican with their associated volumes. Any operator workflow that might
benefit from knowing a volume's encryption_key_id has to read the value from
the Cinder database.

A similar situation applies to backups of an encrypted volume. These backups
include a clone of the volume's encryption key, and the backup's
encryption_key_id is not available via the API.


Use Cases
=========

1. An operator wishes to back up and export Cinder volumes and Barbican
secrets for disaster recovery (DR), and later restore the volumes. Knowing
each encrypted volume's encryption_key_id will facilitate restoring the
correct encryption secret.

2.1 A user or admin wants to identify the Barbican encryption keys used by
Cinder volumes and/or backups. This proposal will allow them to iterate over
the list of items, and note the encryption_key_id when present in its details.

2.2 A user or admin wishes to know whether a Barbican secret is used by
Cinder. This is similar to 2.1, but more from Barbican's perspective. Given
a list of Barbican secrets, the user or admin may want to now which service
is using the secret.


Proposed change
===============

This proposal would add a microversion to include the encryption_key_id field
in the volume and backup details. The field would be included in the datails
only when relevant:

* The encryption_key_id is set (not null)

* It isn't the all-zeros value used by the obsolete fixed-key ConfKeyManager

Alternatives
------------

1. Operators continue to resort to accessing the Cinder database whenever they
   need to know a volume's or backup's encryption_key_id.
2. The proposed change could be enhanced to require admin privileges.

Data model impact
-----------------

None

REST API impact
---------------

A new microversion will be created, and the encryption_key_id will be
added to the volume and backup details.

Security impact
---------------

On one hand, the proposed change will allow users to see the Key Manager
(Barbican) ID where the encryption secret is stored. But it's important to
bear these factors in mind:

* The encryption_key_id is just a UUID, and access to the actual secret is
  protected by Barbican's security policies.

* A volume's encryption_key_id is already present in the connection_info
  returned by the volume attachment API.

Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

The feature will not affect users beyond the fact that displaying the details
of an encrypted volume or backup will include the encryption_key_id field.

The feature does not require changes in cinderclient, openstackclient or
openstack-sdk.

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
  Alan Bishop <abishop@redhat.com>

Work Items
----------

* Add a new microversion.
* Update the API documentation and code.
* Add unit tests.


Dependencies
============

None


Testing
=======

Unit tests provide sufficient coverage, and there are no plans for tempest
changes.


Documentation Impact
====================

Update the API documentation.


References
==========

None

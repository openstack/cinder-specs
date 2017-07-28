..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Support backup import on another Storage database
====================================================================

URL of launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/support-backup-import-on-another-storage-database

* This backup service can use a backup extra metadata in order
  to import a backup on a different Block Storage.

Problem description
===================

* Currently a volume backup can only be restored on the same Block Storage
  service.
  This is because restoring a volume from a backup requires metadata available
  on the database used by the Block Storage service.

* In order to import backup metadata on another Block Storage database
  (i.e disaster recovery site), one has to save the metadata of a volume
  backup available on the source database, and then replicate it together
  with the data to the other Block Storage site.

* Combination of the local backup service that exports and stores
  this metadata, together with replication to another Block Storage site,
  allows you to completely restore the backup even in the event of
  a catastrophic database failure.

* In addition, having a backup metadata together with a volume backup,
  also provides volume portability.
  Specifically, backing up a volume and exporting its metadata will allow you
  to restore the volume on a completely different Block Storage database,
  or even on a different cloud service.

Use Cases
=========

* When a user wants to save backup metadata together with volume backup for
  volume portability purpose, or for replication purpose to disaster
  recovery site, and be able to restore a volume from a backup on the other
  Block Storage site.

Proposed change
===============

* In order to support backup import on a different Block Storage database,
  we need to extend chunked backup driver:

  *  Enabling to save backup metadata together with the data.
  *  Add a cinder client api command, that will parse backup metadata from
     backup_metadata file, and import that metadata to the other
     Block Storage database.
  *  User will be able to restore a volume backup on the other
     Block Storage, using cinder backup restore api.

Alternatives
------------

Support only storage on local storage.
Use slow manual backup and restore methods.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ronen-mesonzhnik

Other contributors:
  None

Work Items
----------

- Implement get_extra_metadata that will return backup's corresponding
  database information as encoded string metadata.
- Implement a cinder client api command that can take a backup metadata
  file, and do the import from it.

Dependencies
============

None.

Testing
=======

None.

Documentation Impact
====================

* In addition to the existing command 'cinder backup-import <METADATA>',
  there will be a command that can accept a file:
  'backup-import-record-from-backup-metadata-file <file_path>'

References
==========

* Link to Export and import backup metadata documentation:
  http://docs.openstack.org/admin-guide-cloud/blockstorage_volume_backups_export_import.html

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Backup Multi-Backends Configuration
===================================

https://blueprints.launchpad.net/cinder/+spec/backup-backends-configuration

Add a new config section for backup configuration options to unify the way how
we do it with volume drivers and support multiple backup drivers in one
service.

Problem description
===================

Right now we've got all backup-specific configuration options defined in the
[DEFAULT] section::

    [DEFAULT]
    backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
    backup_ceph_user = cinder-backups
    backup_ceph_pool = volume-backups
    ...

It leads to mix of backup-specific config options and Cinder general
configuration. This approach also doesn't support backups multi-backend.
Operators can configure multiple backup backends only by configuring several
cinder-backup services.

Current approach blocks Generic backup implementation because drivers
self.configuration won't be able to see required configuration which are
defined in the [DEFAULT] section.

Use Cases
=========

The main use-case here is to decouple backup-specific configuration from the
[DEFAULT] section and introduce a backup_type.

Proposed change
===============

The new config option called 'enabled_backup_backends' will contain a
list of enabled backup backends. This will implement the same approach as we
have for volumes with 'enabled_backends' config option.

All backup-related configuration will be moved into the new backend-specific
section::

    [DEFAULT]
    enabled_backup_backends = ceph_backup, ceph_backup_ssd
    default_backup_type = ceph-backup-type
    ...

    [ceph_backup]
    backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
    backup_ceph_user = cinder-backups
    backup_ceph_pool = volume-backups

    [ceph_backup_ssd]
    backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
    backup_ceph_user = cinder-backups
    backup_ceph_pool = volume-backups-ssd

The old-style backup configuration will be supported at least for one release
to follow all deprecation policies.


With multi-backups configuration we want to introduce backup types too. It
will help us to use all multi-backend advantages. Backup types will follow the
same approach as volume types have:

* __DEFAULT__ backup type will be introduced and all existing backups will be
  migrated to this backup type.
* `default_backup_type` config option will be introduced.
* Backup types will have own extra specs.


Alternatives
------------

Leave everything as-is and introduce some hacks in the code to get volume
drivers working as a backup drivers in the scope of Generic Backup feature.

Data model impact
-----------------

New tables for backup types will be introduced. We'll follow the same approach
as for volume types which works well for Cinder:

BackupType
+--------------+--------------+
|   Field      |     Type     |
+--------------+--------------+
| created_at   | datetime     |
| updated_at   | datetime     |
| id           | varchar(36)  |
| name         | varchar(255) |
| description  | varchar(255) |
| deleted      | boolean      |
| is_public    | boolean      |
+--------------+--------------+


BackupTypeProjects
+----------------+--------------+
|   Field        |     Type     |
+----------------+--------------+
| created_at     | datetime     |
| updated_at     | datetime     |
| id             | varchar(36)  |
| project_id     | varchar(36)  |
| backup_type_id | varchar(36)  |
| deleted        | boolean      |
+----------------+--------------+

BackupTypeExtraSpecs
+----------------+--------------+
|   Field        |     Type     |
+----------------+--------------+
| created_at     | datetime     |
| updated_at     | datetime     |
| key            | varchar(255) |
| value          | varchar(255) |
| deleted        | boolean      |
+----------------+--------------+

REST API impact
---------------

New API endpoints  will be implemented for backup types. It should be similar
like volume type API.

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

Backup creation procedure will contain few more database calls but it should
not affect performance a lot.

Other deployer impact
---------------------

Operators should configure backups using a new mechanism and migrate from
old-style configuration.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  e0ne

Work Items
----------

* Add a new 'enabled_backup_backends' config option and deprecate old-style
  config
* Modify the backup manager to honor new configuration
* Add some unit tests
* Devstack should be able to configure cinder backups in a new way
* Operator documentation should be updated
* API reference should be updated


Dependencies
============

None


Testing
=======

* Unit tests
* Existing tempest tests will cover new functionality


Documentation Impact
====================

* Update documentation to describe new config option
* New REST API endpoints will be documented


References
==========

* https://blueprints.launchpad.net/cinder/+spec/backup-backends-configuration
* https://specs.openstack.org/openstack/cinder-specs/specs/mitaka/scalable-backup-service.html
* https://specs.openstack.org/openstack/cinder-specs/specs/train/untyped-volumes-to-default-volume-type.html

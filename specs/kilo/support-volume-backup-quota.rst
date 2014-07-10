..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Support Volume Backup Quota
===========================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/support-volume-backup-quota

Provide a mean to control volume backup's quota.

Problem description
===================

Since quota take volumes, snapshots and gigabytes into account, it also needs
to take backup into account:

* Currently Backup create API is not an admin API, users of projects could
  create any number of backups.

* If some evil users create many more big backups to exhaust the free space
  of backup storage back-end, it would cause cinder-backup in the state of
  rejecting service.


Proposed change
===============

By supporting volume backup quota, we need to do the following changes.

* Add the default quota of backup number quota item and backup gigabytes quota
  item.

* Cinder quota update API support backup number quota and backup gigabytes
  quota create and update operation.

* Cinder quota show/defaults API support displaying backup quota info.

* Quota Engine support processing with backup quota item.

    ::

        Class QuotaEngine and Class VolumeTypeQuotaEngine support processing
        with backup quota item.
        Add _sync_backups function to get one project's backup number.
        Add _sync_backup_gigabytes function to get one project's backup
        gigabytes number.
        Add configuration item no_backup_gb_quota, which indicates whether to
        take backup's size into account with backup gigabytes quota.
        Add '_sync_backup': _sync_backups to QUOTA_SYNC_FUNCTIONS.
        Add '_sync_backup_gigabytes': _sync_backup_gigabytes to
        QUOTA_SYNC_FUNCTIONS.

* Backup create/delete routine support dealing with backup number quota and
  backup gigabytes quota.

* Limits API support displaying one project's current used number and maximum
  number of backup number and backup gigabytes info.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

Change the following APIs' return value.

* Cinder quota show/defaults API

  * Current return value::

        {
            "quota_set":
            {
                "gigabytes": 1000,
                "snapshots": 10,
                "volumes_rbd": -1,
                "volumes": 10,
                "gigabytes_rbd": -1,
                "snapshots_rbd": -1,
                "id": "bb2ec78e61d3493da61449f3c69db3f3"
            }
        }

  * New return value::

        {
            "quota_set": {
                "gigabytes": 1000,
                "snapshots": 10,
                "volumes_rbd": -1,
                "volumes": 10,
                "snapshots_rbd": -1,
                "gigabytes_rbd": -1,
                "backups": 10,          # new add return item
                "backup_gigabytes": 1000, # new add return item
                "id": "bb2ec78e61d3493da61449f3c69db3f3"
            }
        }

* Limit index API return value

  * Current return value::

        {
            "limits": {
                "rate": [],
                "absolute": {
                    "totalSnapshotsUsed": 0,
                    "maxTotalVolumeGigabytes": 1000,
                    "totalGigabytesUsed": 5,
                    "maxTotalSnapshots": 10,
                    "totalVolumesUsed": 5,    #new add return item
                    "maxTotalVolumes": 10     # new add return item
                }
            }
        }

  * New return value::

        {
            "limits": {
                "rate": [],
                "absolute": {
                    "totalSnapshotsUsed": 0,
                    "maxTotalVolumeGigabytes": 1000,
                    "totalGigabytesUsed": 5,
                    "maxTotalSnapshots": 10,
                    "totalVolumesUsed": 5,
                    "maxTotalVolumes": 10,
                    "totalBackupsUsed": 0,       # new add return item
                    "maxBackupsUsed": 10,        # new add return item
                    "totalBackupGigabytesUsed": 0, # new add return item
                    "maxTotalBackupGigabytes": 1000, # new add return item
                }
            }
        }

Security impact
---------------

DoS via resource exhaustion of backup resources is prevented.

Notifications impact
--------------------

None.

Other end user impact
---------------------

The return values of Cinder quota show/defaults API and limits API
have changed.

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
  ling-yun<zengyunling@huawei.com>


Work Items
----------

* Implement code that mentioned in "Proposed change".
* Add change API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change" and ensure that Cinder Quota feature works well
while adding support for volume backup.


Documentation Impact
====================

The cinder API documentation will need to be updated to reflect the REST API
changes.


References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Incremental backup improvements for L
=====================================

https://blueprints.launchpad.net/cinder/+spec/cinder-incremental-backup-improvements-for-l

This specification proposes to improve the current incremental backup by adding
is_incremental  and has_dependent_backups flags to indicate the type of backup
and enriching the notification system via adding parent_id to report.


Problem Description
===================

In Kilo release we supported incremental backup, but there are still some
points that need to be improved.
1. From the API perspective, there is no place to show the backup is
incremental or not.
2. User also doesn't know if the incremental backup can be deleted or not,
It's important that Cinder doesn't allow this backup to be deleted since
'Incremental backups exist for this backup'. Currently, they must have a try to
know it. So if there is a flag to indicate the backup can't be deleted or not,
it will bring more convenience to user and reduce API call.
3.Enriching the notification system via reporting to Ceilometer, add parent_id
to report

Use Cases
=========

It's useful for 3rd party billing system to distinguish the full backup and
incremental backup, as using different size of storage space, they could have
different fee for full and incremental backups.

Proposed Change
===============

1.When show single backup detail, cinder-api needs to judge if this backup is
a full backup or not checking backup['parent_id'].
2.If it's an incremental backup, judge if this backup has dependent backups
like we do in process of delete backup.
3.Then add 'is_incremental=True' and 'has_dependent_backups=True/False' to
response body.
4.Add parent_id to notification system.


Data Model Impact
-----------------
None.


REST API Impact
---------------
The response body of show incremental backup detail is changed like this:

::
    {
        "backup": {
                    ......,
                    "is_incremental": True/False,
                    "has_dependent_backups": True/False

                  }

    }

If there is full backup, the is_incremental flag will be False.
And has_dependent_backups will be True if the full backup has dependent
backups.

Security Impact
---------------
None

Notifications Impact
--------------------
Add parent_id to backup notification.


Other End User Impact
---------------------
End user can get more info about incremental backup. Enhance user experience.


Performance Impact
------------------
Because we add an additional judgment for dependent backups. We can eliminate
performance impact by adding index to backup table and counting the number of
dependent backups to make judgment in SQL.


IPv6 Impact
-----------
None.


Other Deployer Impact
---------------------
None.


Developer Impact
----------------
None.


Community Impact
----------------
None.


Implementation
==============

Assignee(s)
-----------
wanghao


Work Items
----------
* Add querying and judging code in cinder-api.
* Add parent_id to notification system.
* Add unit tests.


Dependencies
============
None


Testing
=======
Unit tests are needed to ensure response is working correctly.


Documentation Impact
====================
1. Cloud admin documentations will be updated to introduce the changes: 
http://docs.openstack.org/admin-guide-cloud/content/volume-backup-restore.html

2. API ref will be also updated for backups:
http://developer.openstack.org/api-ref-blockstorage-v2.html


References
==========
None
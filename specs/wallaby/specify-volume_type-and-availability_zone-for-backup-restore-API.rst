..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Specify availability_zone and volume_type for backup restore API
================================================================

https://blueprints.launchpad.net/cinder/+spec/availabilityzone-and-volumetype-for-backup-restore

Add two optional parameters to specify availability_zone and volume_type for
cinder backup restore API to better support backup and restore across
availability zones.

Backup across availability_zones is a typical scenario for disaster recovery:
say, backup driver located in AZ-1 is configured to backup data to backup
backend located in AZ-2.  When AZ-1 crashes, user tries to restore volume from
the backup data located in AZ-2.  It would be useful to provide parameters for
user to specify the availability_zone and the volume_type of the restore
volume.

The availability_zone parameter will enable user to create a restore volume to
a specified availability_zone where the backup data is actually located.  The
geographical locality will enhance the performance for restore copy.

While with the volume_type parameter, the backup restore API can provide more
options for user to create a restore volume, say, with rbd backup backend,
compared with LVM, user could choose to restore to a rbd volume, where restore
will do diff copy instead of full copy and thus enhance restore efficiency.

Problem description
===================

Currently, when user makes a request to restore a backup without volume_id, the
backup-restore API will create a restore volume with default volume_type and
default availability_zone.

As the availability_zone is none, cinder scheduler will randomly pick up one
availability_zone to create this restore volume.  In one typical disaster
recovery scenario described above, restore volume with randomly selected
availability_zone may locate in an availability_zone geographically far away
from backup data.  In this way, restore might fail if the restore side backup
driver could not access backup data across availability_zones.  Even if backup
driver could access backup data remotely, cross-wan IO will cause a performance
penalty.  No matter for the sake of correctness or efficiency, user would like
to have control of specifying the availability_zone of restore volume.

Without volume type option, the restore volume will be created with default
volume type and thus default volume backend.  Since the default volume backend
may not understand the backup data format, the backup driver may not be able to
take the most efficient way to restore from the backup data but will do a full
copy.  It is better to give user more options to enhance restore performance.

Use Cases
=========

There are customers who want to restore a backup in a certain backend and
availability_zone without any restore volume. The customers can specify the
volume_type and availability_zone to restore this backup.

Proposed change
===============

In order to solve this problem, we add two optional parameters to specify
volume_type and availability_zone for the backup-restore API.  The volume_type
parameter is to specify the volume backend of the restore volume.  The
availability_zone parameter is to specify the availability_zone where the
restore volume would locate in.  Thus, user could choose restore volume type
according to backup backend and choose restore volume location geographically
close to the backup data location.  This change will give user more options to
ensure restore correctness and enhance restore performance.

Alternatives
------------

None

Data model impact
-----------------

None


REST API impact
---------------

Modify REST API to restore backup:
  * POST /v3.x/{tenant_id}/backups/{id}/action
  * Add volume_type and availability_zone in request body
  * JSON request schema definition:

    .. code-block:: python

      'backup-restore': {
          'volume_id': volume_id,
          'volume_type': volume_type,
          'availability_zone': availability_zone,
          'name': name}

Security impact
---------------

None

Notifications impact
--------------------

Ensure type and AZ are included in the restore.start notification.

Other end user impact
---------------------

End user will be able to restore a backup with specified volume_type and
availability_zone.

Performance Impact
------------------

This might allow cross-az restore scenarios that weren't possible before,
which might cause network performance degradation.

Other deployer impact
---------------------

The deployer will be able to restore a backup with specified volume_type and
availability_zone.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  alan-bishop

Other contributors:
  <None>

Work Items
----------

TBD


Dependencies
============

None


Testing
=======

Unit tests and tempest tests will be provided.

Documentation Impact
====================

None

References
==========

None

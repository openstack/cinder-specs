..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Backup Notification
==========================================

https://blueprints.launchpad.net/cinder/+spec/backup-notification

This blueprint proposes to add the notification support to the backup
service in Cinder, so that cinder can report the usage status to Ceilometer
in backup create, delete and restore.

Problem description
===================

Cinder is supposed to send the notifications to ceilometer to report the
resource usage status. This notification support has been implemented for
the volume and the volume snapshot, but not the backup.

Proposed change
===============

Create backup notification:

* Send a notification to inform the Ceilometer the backup create begins.

* Send a notification to inform the Ceilometer the backup create ends.

Delete backup notification:

* Send a notification to inform the Ceilometer the backup delete begins.

* Send a notification to inform the Ceilometer the backup delete ends.

Restore backup notification:

* Send a notification to inform the Ceilometer the backup restore begins.

* Send a notification to inform the Ceilometer the backup restore ends.

Progress notification:

* It is possible that some drivers can send a create.progress notifications
  periodically. We make it configurable if the backup driver supports to
  send periodic notifications.

The backup information sent to Ceilometer includes backup id, project id,
user id, available zone, host, display name, creation time, status, volume
id, size, service metadata, service and fail reason.

For the progress notification, an additional data indicating the percentage
of the backup progress will be sent as well.

Alternatives
------------

None.

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

This blueprint will add the notification support for the backup service.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

* The configuration option backup_object_number_per_notification has
  been added to indicate how many chunks or objects have been sent
  to the Ceilometer. It applies to the object or chunk based backup
  service, e.g. Swift, Ceph.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Vincent Hou

Other contributors:
  None

Work Items
----------

* Add the notification for the backup usage when creating a backup
* Add the notification for the backup usage when deleting a backup
* Add the notification for the backup usage when restoring a backup
* Add the progress notification for the backup usage when talking an object
  store as the backup service.

Dependencies
============

None

Testing
=======

* Unit tests will be added for the backup notification calls.

Documentation Impact
====================

* The configuration option to configure the progress notifications
  needs to be added for the backup driver, which supports to send
  this type of notification.

References
==========

* Backup notification blueprint and bug replication design session
  https://blueprints.launchpad.net/cinder/+spec/backup-notification
  https://bugs.launchpad.net/cinder/+bug/1326431


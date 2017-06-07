..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Backup driver initialization
============================

https://blueprints.launchpad.net/cinder/+spec/backup-init

We don't initialize Cinder Backup driver during service startup. It means that
we've got cinder-backup service up and running even if it can't connect to the
backup storage.

Problem description
===================

Cinder backup manager does not verify that backup driver is initialized
successfully. If cinder-backup is started successfully we can create a volume
backup. Such backups always will be in 'error' status and tenant user won't be
able to delete them.

Use Cases
=========

Cinder backup service should be marked as 'down' if it can't connect to the
backup storage.

Proposed change
===============

We should introduce for cinder backups the same mechanism as we've got for
volume manager and drivers:

* Introduce 'init_host' method in backup manager which will be called on
  service startup and verify that backup driver is initialized: verify driver's
  configuration is correct, depends on driver, we could check connection to
  storage, list of available backups. etc.

* If backup driver initialization fails, manager will mark backup service
  as 'down'.

In case of initialization failure, cinder will try to do it periodically
depends on 'periodic_interval' config option value.

Backup service should be initialized in 'service_down_time' time
interval or will be marked as 'down'.

Alternatives
------------

Check for backup storage is available on backup create call. If storage is not
available, remove 'host' field from backup object. We could try to re-schedule
backup creation on the other host.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

New notifications for backup initialization failure and success will be added.

Other end user impact
---------------------

User will be able to delete backup in error state if it was not created on
backend.

Performance Impact
------------------

None

Other deployer impact
---------------------

New 'backup_periodic_initialization' and 'backup_initialization_timeout'
config option will be added. Deployer have to enable
'backup_periodic_initialization' if needed.

Developer impact
----------------

Backup driver developers should implement new APIs.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Ivan Kolodyazhny <e0ne@e0ne.info>

Other contributors:
  Backup drivers maintainers.

Work Items
----------

* Implement 'do_setup' method in a base backup driver wich won't do anything

* Implement 'do_setup' in each backup driver.

* Call driver's 'do_setup' during backup-manager 'init_host' call.



Dependencies
============

None


Testing
=======

Both unit and Tempest tests should be implemented to cover new feature.


Documentation Impact
====================

None


References
==========

* https://bugs.launchpad.net/cinder/+bug/1598709

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Volume Migration Improvement
============================

https://blueprints.launchpad.net/cinder/+spec/migration-improvement

This specification proposes to improve the current volume migration in terms
of implementing a better way to manage the volume migration status,
adding the migration progress indication, enriching the notification system via
reporting to Ceilometer, guaranteeing the migration quality via tempest tests
and CI support, etc. It targets to resolve the current issues for the available
volumes only, since we need to wait until the multiple attached volume
functionality lands in Nova to resolve the issues related to the attached
volumes. There is going to be another specification designed to cover the
issues regarding the attached volumes.

Use Cases
=========

Having the ``well defined`` capabilities will allow the deployer to see what
common capabilities are shared beyond their deployed backends in Cinder.

There are three cases for volume migration. The scope of this spec is for the
available volumes only and targets to resolve the issues within the following
migration Case 1 and 2:

Within the scope of this spec
1) Available volume migration using the "dd" command.
For example, migration from LVM to LVM, between LVM and vendor driver, and
between different vendor drivers.

2) Available volume migration between two pools from the same vendor driver
using driver-specific way. Storwize is taken as the reference example for this
spec.

Out of the scope of the spec
3) In-use(attached) volume migration using Cinder generic migration.

Problem description
===================

Currently, there are quite some straightforward issues about the volume
migration.
1. Whether the migration succeeds or fails is not saved anywhere, which is very
confusing for the administrator. The volume status is still available or
in-use, even if the administrator mentions --force as a flag for cinder migrate
command.
2. From the API perspective, the administrator is unable to check the status of
the migration. The only way to check if the migration fails or succeeds is
to check the database.
3. During the migration, there are two volumes appearing in the database record
via the check by "cinder list" command. One is the source volume and one is the
destination volume. The latter is actually useless to show and leads to
confusion for the administrator.
4. When executing the command "cinder migrate", most of the time there is
nothing returned to the administrator from the terminal, which is unfriendly
and needs to be improved.
5. It is probable that the migration process takes a long time to finish.
Currently the administrator gets nothing from the log about the progress of the
migration.
6. Nothing reports to the Ceilometer.
7. There are no tempest test cases and no CI support to make sure the migration
truly works for any kind of drivers.

We propose to add the management of the migration status to resolve
issues 1 to 4, add the migration progress indication to cover Issue 5, add
the notification to solve Issue 6 and add tempest tests and CI support to
tackle Issue 7.

Proposed change
===============

At the beginning, we declare that all the changes and test cases are dedicated
to available volumes. For the attached volumes, we will wait until the multiple
attached volume functionality get merged in Nova.

* Management of the volume migration status:

If the migration fails, the migration status is set to "error". If the
migration succeeds, the migration status is set to "success". If no migration
is ever done for the volume, the migration status is set to None. The migration
status is used to record the result of the previous migration. The migration
status can be seen by the administrator only.

The administrator has several ways to check the migration status:
1) The administrator can do a regular "volume list" with a filter
"migration_status=<expected volume migration status>" to find all the volumes
with the specified migration status. If no filter is specified, all the volumes
will list the migration status.
2) The administrator can issue a "get" command for a certain volume and the
migration status can be found in the field 'os-vol-mig-status-attr:migstat'.

If the administrator issues the migrate command with the --force flag, the
volume status will be changed to 'maintenance'. Attach or detach will not be
allowed during migration. If the administrator issues the migrate command
without the --force flag, the volume status will remain unchanged. Attach or
detach action issued during migration will abort the migration. The status
'maintenance' can be extended to use in any other situation, in which the
volume service is not available due to any kinds of reasons.

We plan to provide more information when the administrator is running
"cinder migrate" command. If the migration is able to start, we return a
message "Your migration request has been received. Please check migration
status and the server log to see more information." If the migration is
rejected by the API, we shall return messages like "Your migration request
failed to process due to some reasons".

We plan to remove the redundant information for the dummy destination volume.
If Cinder Internal Tenant(https://review.openstack.org/#/c/186232/) is
successfully implemented, we will apply that patch to hide the destination
volume.

* Migration progress indication:

We would like to introduce a poll mechanism to check the migration progress in
a certain interval as the implementation for the migration progress indication.
The poll mechanism can be realized in a loop, and the migration progress will
be checked in a certain interval, which is configurable in cinder.conf. This
mechanism can be running in parallel to the volume migration.

If the volume copy starts, another thread for the migration progress check will
start as well. If the volume copy ends, the thread for the migration progress
check ends.

A driver capability named migration_progress_report can be added to each
driver.
It is either True or False. This is for the case that volumes are migrated
from one pool to another within the same storage back-end. If it is True, the
loop for the poll mechanism will start. Otherwise, no poll mechanism will
start.

A configuration option named migration_progress_report_interval can be added
into cinder.conf, specifying how frequent the migration progress needs to be
checked. For example, if migration_progress_report_interval is set to
30 seconds, the code will check the migration progress and report it every
30 seconds.

If the volume is migrated using dd command, e.g. volume migration from LVM to
LVM, from LVM to vendor driver, from one back-end to another via blockcopy,
etc, the migration progress can be checked via the position indication of
/proc/<PID>/fdinfo/1.

For the volume is migrated using file I/O, the current file pointer is able to
report the position of the transferred data. The migration progress can be
checkedvia this position relative to EOF.

If the volume is migrated within different pools of one back-end, we would like
to implement this feature by checking the stats of the storage back-end.
Storwize V7000 is taken as the reference implementation about reporting the
migration progress. It is possible that some drivers support the progress
report and some do not. A new key "migration_progress_report" will be added
into the driver to report the capability. If the back-end driver supports to
report the migration progress, this key is set to True. Otherwise, this key is
set to False and the progress report becomes unsupported in this case.

The migration progress can be checked by the administrator only. Since the
progress is not stored, each time the progress is queried from the API, the
request will be scheduled to the cinder-volume service, which can get the
updated migration progress for a specified volume and reports back.

Alternatives
------------

We can definitely use a hidden flag to indicate if a database row is displayed
or hidden. However, cinder needs a consistent way to resolve other issues like
image cache, backup, etc, we reach an agreement that cinder internal tenant is
the approach to go.

The purpose that we plan to have a better management of the volume migration
status, add migration progress indication, report the stats to Ceilometer and
provide tempest tests and CI, is simply to guarantee the migration works in a
more robust and stable way.


Data model impact
-----------------

None


REST API impact
---------------

The REST API should be able to provide the migration status and the migration
progress information for the volumes. For the migration status, it can be
retrieved from the database. For the migration progress, the API request
will be scheduled to the cinder volume service, where the volume is located,
and cinder volume service reports the updated progress back.


Security impact
---------------

None


Notifications impact
--------------------

The volume migration should send notification to Ceilometer about the start,
and the progress and the finish.


Other end user impact
---------------------

None


Performance Impact
------------------

None


Other deployer impact
---------------------

If the back-end driver supports the migration progress indication, a new
configuration option migration_progress_report_interval can be added. The
administrator can decide how frequent the cinder volume service to report the
migration progress. For example, if migration_progress_report_interval is set
to 30 seconds, the cinder volume service will provide the progress information
every 30 seconds.


Developer impact
----------------

The driver maintainer or developer should be aware that they need to add a new
capability to indicate whether their driver support the progress report.
If yes, they need to implement the related method, to be provided in the
implementation of this specification.

If their drivers have implemented volume migration, integration tests and
driver CI are important to ensure the quality. This is something they need to
pay attention and implement for their drivers as well.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Vincent Hou (sbhou@cn.ibm.com)

Other contributors:
  Jay Bryant
  Jon Bernard


Work Items
----------

* Management of the volume migration status:

1) Change the migration_status to "error" if the migration fails; Change the
migration_status to "success" if the migration succeeds.
2) Change the volume status to "maintenance" if the administrator executes
the migration command with --force flag. No attach or detach is allowed during
this migration. If the administrator executes the migration command without
--force flag, the volume status will stay unchanged. Attach or detach during
migration will terminate the migration to ensure the availability of
the volume.
3) Enrich cinderclient with friendly messages returned for cinder migrate and
retype command.
4) Hide the redundant dummy destination volume during migration.

* Migration progress indication:

Add a loop to wrap the implementation of the poll mechanism.

The driver, which supports the migration progress report, will set
migration_progress_report to True. Otherwise, set it to False.

The option migration_progress_report_interval will be used to specify the time
interval, in which the migration progress is checked.

1) If the volume is migrated between LVM back-ends, or one back-end to another,
the position indication of /proc/<PID>/fdinfo/1 can be checked to get the
progress of the blockcopy.

2) If the volume is migrated within different pools of one back-end, we would
like to check the progress report of the back-end storage in a certain time
interval.

The migration percentage will be logged and reported to Ceilometer.

* Notification:

Add the code to send the start, progress and end to Ceilometer during
migration.

* Tempest tests and CI support:

This work item is planned to finish in two steps. The first step is called
manual mode, in which the tempest tests are ready and people need to configure
the OpenStack environment manually to meet the requirements of the tests.

The second step is called automatic mode, in which the tempest tests can run
automatically in the gate. With the current state of OpenStack infrastructure,
it is only possible for us to do the manual mode. The automatic mode needs to
collaboration with OpenStack-infra team and there is going to be a blueprint
for it.

The following cases will be added:
1) From LVM(thin) to LVM(thin)
2) From LVM(thin) to Storwize
3) From Storwize to LVM(thin)
4) From Storwize Pool 1 to Storwize Pool 2

Besides, RBD driver is also going to provide the similar test cases from (2)
to (4) as above.

We are sure that other drivers can get involved into the tests. This
specification targets to add the test cases for LVM, Storwize and RBD drivers
as the initiative. We hope other drivers can take the implementation of LVM,
Storwize and RBD as a reference in future.

* Documentation:

Update the manual for the administrators, and the development reference for
the driver developers and maintainers.


Dependencies
============

Cinder Internal Tenant: https://review.openstack.org/#/c/186232/
Add support for file I/O volume migration:
https://review.openstack.org/#/c/187270/


Testing
=======

Depending on ability to parse the required information for the LVM driver, the
following scenarios for available volumes will taken into account:
1) Migration using Cinder generic migration with LVM(thin) to LVM(thin).
2) Migration using Cinder generic migration with LVM(thin) to vendor driver.
3) Migration using Cinder generic migration with vendor driver to LVM(thin).
4) Migration between two pools from the same vendor driver using
driver-specific way.

There are some other scenarios, but for this release we plan to consider the
above.
For scenarios 1 to 3, we plan to put tests cases into Tempest.
For Scenario 4, we plan to put the test into CI.
The reference case for Scenario 2 is migration from LVM to Storwize V7000.
The reference case for Scenario 3 is migration from Storwize V7000 to LVM.


Documentation Impact
====================

Documentation should be updated to tell the administrator how to use the
migrate and retype command. Describe what commands work for what kind of use
cases, how to check the migration status, how to configure and check
the migration indication, etc.

Reference will be updated to tell the driver maintainers or developers how to
change their drivers to adapt this migration improvement via the link
http://docs.openstack.org/developer/cinder/devref/index.html.

References
==========

* https://blueprints.launchpad.net/cinder/+spec/migration-improvement
* https://etherpad.openstack.org/p/volume-migration-improvement

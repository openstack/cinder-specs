..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Restrict new container creation for Cinder Backup
=================================================

The URL of launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/restrict-backup-container-creation

This spec is designed to allow operators to restrict new container creation
by users during backup creation, when a non-existent container is passed as
``container`` argument with a POST ``/v3/{project_id}/backups`` request.


Problem description
===================

While creating Cinder backups, users are able to supply an arbitrary container
name, where the backup will be stored. While this behavior makes total sense
with the Swift driver and ``backup_swift_auth=per_user``, as this will utilize
tenant quota and data will be accounted properly, using arbitrary containers
might be unwanted by operators while using other drivers.

Current behavior may cause various negative side-effects when users use
arbitrary container names, for example: being unable to enforce storage policies,
going out of designed quota for backups, complications for accounting of
consumed disk space and billing, etc.

At the moment container creation is supported by the following drivers:

* Swift
* S3
* GCS
* POSIX


Use Cases
=========

For instance, with the S3/GCS driver, as an operator, I want to pre-create a
series of buckets with different policies like object locking with various
retention rules or different storage classes. While current behavior might
work, ability for users to create new buckets will result in an approach that
is not error-prone, as due to typos backups may end in unexpected locations.

With the POSIX driver as example, as operator I want to have different mounts and
underlying storages per "container". With current behavior this approach is
impossible as users may bypass the defined mount boundaries by creating a new
path with an arbitrary container name.

Proposed change
===============

It is proposed to introduce a new configuration option
``backup_create_containers``, which will control whether Cinder Backup will
attempt to create a new container when a user has supplied a non-existing
container name in request or fail such request.

When this option is set to False, duty of creating all containers, including
the default one, falls under operator responsibility. This means that all
containers, including the default one, must be pre-created before any backup
can be done.

Alternatives
------------

Alternative approach would be to introduce a separate policy, which may allow
users with required privileges to create containers with backups.
However, implementation of this approach is more complex while not providing
much more benefits for the use case, given that operators are usually not
executing backup creation on their own.

Data model impact
-----------------

No data model impact

REST API impact
---------------

No direct impact on REST API.

POST requests to ``/v3/{project_id}/backups`` supplying ``container``
parameter that is not found on backends with
``backup_create_containers=False`` will result in backup creation failures,
with the corresponding reason being recorded in ``fail_reason`` column of
``backups`` table.

Security impact
---------------

This change should positively impact overall security of Cinder Backup,
as it allows restricting potentially too open possibilities of container
creation, which depending on deployment design and driver may lead to
unexpected behavior or even denial of service.

Active/Active HA impact
-----------------------

Not applicable for Cinder Backup service


Notifications impact
--------------------

No impact

Other end user impact
---------------------

When ``backup_create_containers=False`` users will be able to create backups
only in a predefined set of container names provided by the cloud provider and
will not be able to use arbitrary container names.

A disadvantage of the approach, is that check for container existence is
performed asynchronously by cinder-backup service, so user will not receive
an error on issuing API request, but will see a 404 error as a reason for
backup creation failure instead.
The error will be shown in backup failure reason and not in the API response
to the request, as API does not wait for RPC messaging with the backup
service to happen.

There is also no discoverability mechanism if this property is enabled or not, so user
is not able to know in advance if ``backup_create_containers`` is enabled or
not. It is the cloud provider responsibility to inform users about cases, where
``backup_create_containers`` is set to ``False``.

Performance Impact
------------------

No performance impact

Other deployer impact
---------------------

Default value of ``backup_create_containers`` is proposed to be set to
``True``, to preserve existing behavior.
Therefore, no deployer impact is expected.

Developer impact
----------------

No impact


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  Dmitriy Rabotyagov <noonedeadpunk>

Work Items
----------

* Introduce new configuration parameter for cinder-backup globally
* Add support to S3 backup driver
* Add support to GCS backup driver
* Add support to Swift backup driver
* Add support to POSIX backup driver


Dependencies
============

No dependencies


Testing
=======

Unit testing of this option will be introduced, to ensure that cinder-backup
will not call for new container creation to the backend, when
``backup_create_containers`` is set to ``False``.


Documentation Impact
====================

New option will be documented as part of the Cinder Backup configuration
guide.


References
==========

* Reference implementation: https://review.opendev.org/c/openstack/cinder/+/959425

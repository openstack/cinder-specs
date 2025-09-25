..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Backup container name templating
================================

The URL of launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/backup-container-name-template

This spec is focused on introducing the ability to define a container name
template, by extending user-supplied or default container names with variables,
like project or user ID, volume ID, etc.

As a result, the currently defined ``backup_default_container`` variable should
be a required parameter in the resulting template.
The container template name is defined in cinder-backup configuration file, and is
an admin feature to adjust resulting container name according to the template.


Problem description
===================

For backup drivers based on ``chunkeddriver`` a default ``container`` name is
supplied in the configuration file, while users are able to supply any
arbitrary ``container`` name for backup creation request.

While this provides some level of flexibility and lets users separate data in
a way, current implementation is not suitable for public cloud offerings and
does not provide any efficient or guaranteed tenant separation on the storage
side, as all tenant data will be stored in the same bucket by default, with
various deviations in case of user-supplied container names.

At the moment, this applies to most of the backends which are based on the
``chunkeddriver``, specifically:

* S3
* GCS
* POSIX
  * NFS
  * GlusterFS

This issue is not relevant for the ``swift`` backend, which is also a child
of ``chunkeddriver``, but supports proper multi-tenancy and the only driver
which allows storing the data within the tenant quota.


Use Cases
=========

* Operators might want to ensure that data is stored in different buckets or
  containers for each tenant.
* Set different quotas or policies on the backend for different tenants.
* Ensure that billing policies are adequate and possible based on container
  names.
* Diversify risks in case of bucket metadata corruption in third-party S3
  implementations (like Ceph RadosGW).
* Use different mount points based on tenant.

Proposed change
===============

Implement a configuration option ``backup_container_name_template`` which
will be defined for the ``chunkeddriver``. Having a single option simplifies
the logic and makes code more readable.

Default value for the option will be equal to the ``%(backup_default_container)``
which is the default name defined for respective drivers.

Although the option is defined per driver, logic for processing it
will be implemented centrally inside the ``chunkeddriver``.

It is proposed to implement template functionality similar to
``backup_name_template``, with the exception to allow operators to use
wider range of variables to use as part of the template.

Among such variables it is proposed to allow using:

* backup_default_container (required)
* backup_id
* project_id
* domain_id
* user_id
* volume_id
* region_name
* availability_zone

Defined ``backup_default_container`` parameter is a required parameter
in the template. ``cinder-backup`` must validate supplied templates
at startup and fail if the configuration is incorrect or missing
required ``backup_default_container`` in the template.

Optionally supplied via API ``container_name`` during backup creation
is passed to the ``chunkeddriver`` as ``backup_default_container``,
thus introduced template will cover not only the default container name
defined in the config, but also any arbitrary ones supplied by users.

In order to avoid end-user confusion by responding with a different container
name, then user asked for, we should avoid exposing resolved container name
to the end user.
As a result, actual container name is not written down to the database, so
templating is executed during both backup and restoration process only in
memory.

As a side effect, it is non-trivial to change the value of
``backup_container_name_template`` once backups are already created.
Responsibility to ensure that all existing containers are renamed
according to the new value of the ``backup_container_name_template``
is offloaded to operators, whenever they decide to change an existing
value.

Alternatives
------------

Alternatively we can add multiple configuration options, depending on the
driver naming convention. This approach may be preferred to keep consistency
in option naming and be rendered closer to each other in config reference
and sample configuration files.
However, it adds unnecessary complexity to the code.

In this case, the proposed list of configuration option names can be:

* Swift: ``backup_swift_container_name_template``
* S3: ``backup_s3_store_bucket_name_template``
* POSIX: ``backup_container_name_template``
* GCS: ``backup_gcs_bucket_name_template``

Data model impact
-----------------

No impact

REST API impact
---------------

No impact

Security impact
---------------

No security impact is expected, as the plan is to have a well-defined list
of variables available for substitution in the template, instead of giving
full access to the context and available data in there.

Active/Active HA impact
-----------------------

Not related to cinder-volume service

Notifications impact
--------------------

No impact

Other end user impact
---------------------

As changes in container name and template will not be exposed to the end-users,
and name replacement will happen during runtime without writing it to the
database, no end user impact is expected.

Performance Impact
------------------

All data required for template rendering should be already present in
context while calling
``cinder.backup.chunkeddriver.ChunkedBackupDriver.update_container_name``
so no additional calls or performance impact is expected.

Other deployer impact
---------------------

Deployers will be able to leverage new container name templates for
fine-grained control over location and naming convention of resulting
container names, allowing them to use different containers based on
regions, availability zones, tenants, etc.

In case a deployer decided to change the value of ``backup_default_container``
they will have to also rename all already existing containers in order
to match the value. Otherwise, restoration of backups created with
different value will fail with 404 Not Found.

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

* Remove noop overrides of ``update_container_name`` method in drivers
* Add configuration parameters to the respective drivers
* Implement templating on backup create and restore process

Dependencies
============

No dependencies


Testing
=======

Unit testing of new options will be introduced, to ensure that cinder-backup
will apply the template to provided/stored container name on backup
creation/restoration.
We also need to test that only supported values can be defined and used in
the template, as well as required ``backup_default_container`` is always part
of the template.

Documentation Impact
====================

New option will be documented as part of the Cinder Backup configuration
guide.


References
==========

* Reference implementation: https://review.opendev.org/c/openstack/cinder/+/962909

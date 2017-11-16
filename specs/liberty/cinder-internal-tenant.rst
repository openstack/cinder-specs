..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cinder internal tenant
==========================================

https://blueprints.launchpad.net/cinder/+spec/cinder-internal-tenant

This spec proposes the addition of config options and internal plumbing
required for a cinder-internal tenant. This tenant can then be used by Cinder
as the owner of internal objects that we don't really want owned by any
normal users.

Problem description
===================

There are several use cases for 'internal' cinder objects where we would like
to prevent a normal user from seeing them. The approach that seems to have
the most backing is to a special tenant that cinder can use for owning these
internal volumes/snapshots/etc.


Use Cases
=========

There are a few different use cases for the internal tenant. The generic use
case is for any volume or snapshot object in Cinder that we do not want
exposed to normal users, but would like to keep track of and have treated as
normal volumes and snapshots.

There are a few features which would be able to take advantage of this:

* Image Caching - Volumes (and potentially snapshots) that are used as part
  of the image cache could be owned by the tenant.
* Volume Migration - Temporary volumes while migrations are taking place could
  be owned by the cinder-internal tenant.
* Non-Disruptive Backups - Temporary snapshots and volumes could be owned by
  the cinder-internal tenant.

Proposed change
===============

I think initially the easiest way to make this work is to add new config
options that take in the credentials for the tenant. This would require an
admin to create and configure the tenant manually and then set the values in
cinder.conf. These config options will be:

* cinder_internal_tenant_username
* cinder_internal_tenant_password

These config options will default to 'None'. For some features like the image
cache this isn't such a big deal. If the credentials are none, and the tenant
is unavailable the cache just won't actually create cache objects and things
fall back to the original functionality. It will however be more difficult for
features like migration or backups where they may not be able to fall back
nicely to a behavior that does not need the credentials.

Things like quota and other permissions would be managed by an admin only,
cinder will not modify things automatically. In the future it would be
possible to have default values for the config options, and allow cinder
to mange the tenant. That functionality will be out of scope for this initial
implementation.

For the initial change I don't think we should keep any of this info in the
cinder db. Everything comes from the config file and is re-checked at startup.

Any usage of the tenants objects should assume they could be deleted or
cleaned up at any time, I don't think this is any different than the case
today where a user could delete a second volume halfway through migration
though. So this is not a regression. The idea behind this being that as an
admin you could periodically clean these objects up if they start to
accumulate, but you wouldn't really need to worry about which ones are 'safe'.

Alternatives
------------

Another way to approach this problem is to add 'hidden' flags to volumes and
snapshots. The biggest downside of this is that all API's need to then know
about this and start filtering, and potentially clients and commands need
new '--hidden' or '--show-all' kind of flags. In addition these hidden volumes
may either take quota from a real user or not be accounted for at all.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Care will need to be taken to ensure the tenant credentials are not exposed
to anyone other than cloud administrators and that users cannot create volumes
or other objects as the internal tenant.

Notifications impact
--------------------

None

Other end user impact
---------------------

This change by itself shouldn't have any end user impact. Things built on top
of it may change what a user sees while using commands like migrate.

Performance Impact
------------------

None

Other deployer impact
---------------------

New config options and setup steps when configuring cinder.

Additional documentation to read.

Developer impact
----------------

New internal API's to use to create objects as the internal tenant.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  patrick-east

Other contributors:
  None

Work Items
----------

* Add in support for the new config options.
* Add in utility functions to get the tenant information and anything else
  required to use the internal tenant for operations.
* Possibly add support in to DevStack to configure this by default for future
  development on top of the feature.
* Documentation on how to create and configure the tenant.

Dependencies
============

None

Testing
=======

Unit tests for the utility functions.

This change by itself won't require tempest test changes or additions, but
features utilizing it will.

Documentation Impact
====================

* New instructions for how to create and configure the tenant in the
  installation instructions.

* http://docs.openstack.org/trunk/config-reference/content/
* http://docs.openstack.org/user-guide-admin/

References
==========

* http://eavesdrop.openstack.org/meetings/cinder/2015

  /cinder.2015-05-27-16.00.log.html

* https://blueprints.launchpad.net/cinder/+spec/image-volume-cache


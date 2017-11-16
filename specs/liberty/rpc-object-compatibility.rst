..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
RPC and VersionedObject Compatibility
==========================================

https://blueprints.launchpad.net/cinder/+spec/rpc-object-compatibility

The following proposal is to enable rolling upgrade in cinder.  More
specifically, it allows cinder services (cinder-api, cinder-scheduler,
cinder-backup, and cinder-volume) to be updated one at a time and yet still
be operational (only one service is taken down for upgrade instead of all).
It follows the rolling upgrade feature in nova, using oslo_versionedobjects,
but is different in that there is no indirection API.  Instead, RPC versions
and objects are going to be pinned to a backwards compatible version and
later switched to the latest version when all services are upgraded.


Problem description
===================

Today, when a new milestone is released, it can take quite a while to upgrade
all the service within an OpenStack cloud to the latest release.  Upgrade
involves: synching the database to the latest schema and updating each
distributed cinder service.  This process can take down part of a cloud for a
few hours or possibly longer.  Operators ideally like to reduce the down time
for their cloud when upgrading to the next release.

Use Cases
=========

As an operator, I want to individually upgrade each service to the latest
release, but not take down my entire cloud in the process.

Proposed change
===============

There are a couple of problems with upgrading a distributed service such as
cinder:
1. The content sent over RPC can change.
2. The RPC interface itself can change, i.e. new keyword arguments.

To address these problems, the content can be made backwards compatible using
oslo_versionedobject, and the RPC interface can be pinned to a specific
version until all components within cinder are upgraded.

The implementation is as follows:
1. When a service starts, it registers the RPC and object versions it knows
about, i.e. minimum and latest versions.

2. On upgrade, the admin upgrades each individual cinder service.  When the
service starts, it knows that there is a newer version, but continue to run
at the current/pinned version indicated in the database.  The database is
the source of truth for which version a service should run.

There is no specific order for which cinder service should be upgraded first.
The database keeps track of the RPC and object versions that the service
should be using.  The service will continue to use the specific versions
until an all clear is given by the admin (see below).

3. While other cinder services are being upgraded, the service has to
constantly check the database for the current and available RPC and object
versions.  Since constantly accessing the database is a major performance hit,
it can be minimized by caching a copy of the table tracking the versions.
The copy will be valid for a few seconds (e.g. 5 seconds), and it is refreshed
afterwards.  This at least brings down the number of database access for each
RPC call.

A flag is used internally within the service to track if it should
run in backwards compatibility mode (e.g. BACKWARD_COMPAT_MODE), i.e. RPC and
object versions to be made backwards compatible.  If the current version is
lower than the available version, then BACKWARD_COMPAT_MODE is set to true.
Otherwise, it is set to false.  There is logic in each service's rpcapi.py
to set the correct keyword arguments for a given RPC version.  There is also
logic in the objects being sent over RPC to set the correct attributes for
a given target version.  More specifically, each object should be made
backwards compatible by a call to it's own obj_make_compatible(), which
transforms the object to a given target version.

4. Once all cinder services are upgraded, the admin executes a
"cinder-manage version upgrade" to switches the current/pinned versions to
the latest available versions.  This command basically updates the
service_versions table in the cinder database.  Since the current version is
equal to the available version, BACKWARD_COMPAT_MODE is set to false and each
RPC call no longer needs to check the database.

By using the BACKWARD_COMPAT_MODE flag, each cinder service only needs to
be restarted once (i.e. once for upgrade).  Once the flag is turned off, each
cinder service can dynamically switch over to using the new RPC and object
versions.

Once most of the cinder internals are switched to use oslo_versionedobject and
the RPC compatibility layer (this feature) is added, rolling upgrade should
be supported (hopefully in the Liberty release and onwards).  Rolling upgrade
should be limited to two releases older.  For example, release N would be
backwards compatible with release L (Liberty) and M, and release O would be
backwards compatible with release M and N.  For a host/service that is too
old, it should raise an exception on startup and not be allowed to start.

Alternatives
------------

The upgrade process can be left as-is, where all the components must be taken
down and upgraded at the same time.  Some cloud operators, e.g. Rackspace,
have automated their upgrade procedure to a few minutes.  However, other cloud
operators have a need for this feature.

Data model impact
-----------------

A new table, called service_versions, would be created to track the RPC and
object versions.  The schema would be as follows:

.. code-block:: python

  service_versions = Table(
      'service_versions', meta,
      Column('created_at', DateTime(timezone=False)),
      Column('updated_at', DateTime(timezone=False)),
      Column('deleted_at', DateTime(timezone=False)),
      Column('deleted', Boolean(create_constraint=True, name=None)),
      Column('id', Integer, primary_key=True, nullable=False),
      Column('service_id', String(length=255)),
      Column('rpc_current_version', String(length=36)),
      Column('rpc_available_version', String(length=36)),
      Column('object_current_version', String(length=36)),
      Column('object_available_version', String(length=36)),
      mysql_engine='InnoDB'
  )

The service_id is service name + host.  The \*_current_version is the version
a service is currently running and pinned at.  The \*_available_version is the
version a service knows about and (if newer) could upgrade to.

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

A new command, i.e. "cinder-manage version upgrade", would be introduced to
switch the cinder services to use the latest RPC and object versions.

Performance Impact
------------------

During upgrade, before a call is sent over RPC, it would have to check if
it needs to be backwards compatible.  If so, it would need to massage the RPC
interface and object to be backwards compatible.  It would incur a cost on
performance because there would be extra database calls to find the current
and available RPC and object versions.  Since accessing the database before
each RPC call is a major performance hit, it can be minimized by caching a copy
of the table tracking the versions.  The copy will be valid for a few seconds
(e.g. 5 seconds), and it is refreshed afterwards.  This at least brings down
the number of database access for each RPC call.  Once upgrade is done and
all services are upgrade to the latest versions, there would no longer be a
need to check the database.

Other deployer impact
---------------------

None

Developer impact
----------------

Any new changes to the RPC interface within cinder-api, cinder-scheduler,
cinder-volume, or cinder-backup would have to add to the backwards compatible
layer from when this feature merges and onwards.  Also, any new changes to an
object, e.g. volume, snapshot, etc., must be made backwards compatible in the
object's obj_make_compatible().


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thang-pham

Other contributors:
  DuncanT (who thought of the feature)

Work Items
----------

* Create service_versions table to track RPC and object versions.

* Register each cinder service's RPC and object versions on startup.

* Create RPC compatibility layer in each cinder component's rpcapi.py to
  massage the object and RPC interface before it is sent over RPC.

* Create a "cinder-manage version upgrade" CLI to switch each cinder service
  to use the latest versions.


Dependencies
============

* oslo_versionobjects for volumes, backups, service, consistency_group,
  quota need to be merged so that objects can be made backwards compatible.


Testing
=======

Ideally, there should be a rolling upgrade test within tempest (e.g. grenade)
to test basic RPC and object pinning between different releases.  However,
such testing would only apply to release M and onwards because most of the
oslo_versionedobject and RPC compatibility layers are not in previous
releases.


Documentation Impact
====================

It should be documented that operators can upgrade cinder components
individually, without taking down the entire cloud.  At the end of the
process, a "cinder-manage version upgrade" must be executed to switch the
services to use the latest RPC and object versions.


References
==========

* Etherpad: https://etherpad.openstack.org/p/cinder-rolling-upgrade

* Versioning prototype: https://review.openstack.org/#/c/184404/

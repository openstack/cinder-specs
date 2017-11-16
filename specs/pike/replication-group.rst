..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Replication with More Granularity (Tiramisu)
============================================

https://blueprints.launchpad.net/cinder/+spec/replication-cg

In Mitaka, replication v2.1 (Cheesecake) [1][2] was implemented to support
a disaster recovery scenario when an entire host can be failed over to
a secondary site. The Cheesecake API was admin facing.

At the Mitaka mid-cycle meetup, Cinder team decided that after Cheesecake
is implemented, Tiramisu will be the next step. Tiramisu is the code name
for tenant facing replication APIs with more granularity. It gives tenant
more control of what should be replicated together.

Problem description
===================

Replication v2.1, which was implemented in Mitaka as an admin API, was
designed to solve a true DR scenario when the entire backend has to be
failed over to the backend on the secondary site. This is definitely a
very important use case.

There are other cases where the customers also want to have more control
over the granularity of replication. The ability to replicate a group
of volumes and test failover without affecting the entire backend is
also a desired feature. A tenant facing API is needed to meet this
requirement because only a tenant knows what volumes should be grouped
together for his/her application.

It is true that you can do group based replication with Cheesecake by
creating a volume type with replication information and creating volumes
of that volume type so all volumes created with that type belongs to
the same group. So what can Tiramisu do that Cheesecake cannot?

In Tiramisu, a group construct will be used to manage the group of
volumes to be replicated together for the ease of management.

* The group constuct designed in the Generic Volume Group spec [3] will
  provide tenant facing group APIs to allow a tenant to decide what
  volumes should be grouped together and when and what volumes
  should be added to or removed from the group.

* Enable replication function will be added to allow the tenant to
  decide when all volumes needed for a particular application have
  been added to the group and signal the start of the replication
  process.

* Disable replication function will be added to allow the tenant to
  decide when to stop the replication process and make adjustments
  to the group.

* Failover replication function will be added for the tenant to
  test a failover of a group and make sure that data for the
  application will not be compromised after a failover.

* A tenant can test a failover without affecting the entire backend.

* A tenant can decide when and whether to failover when the disaster
  strikes.

If a tenant has already failed over a group using the Tiramisu API
and then a true DR situation happens, the Cheesecake API can still
be used to failover the entire backend. The group of volumes already
been failed over should not be failed over again. Tiramisu will be
using the config options such as ``replication_device`` designed for
Cheesecake.

Use Cases
=========

* An application may have a database stored on one volume and logs stored on
  another and other related files on other volumes. The ability to replicate
  all the volumes used by the application together is important for some
  applications. So the concept to have a group of volumes replicated
  together is valuable.

* A tenant wants to control what volumes should be grouped together during
  a failover and only the tenant has knowledge on what volumes belonging to
  the same application and should be put into the same group.

* A tenant can determine when to start the replication process of a group
  by doing enable replication after all volumes used by an application
  have been added to the group.

* A tenant can determine when to stop the replication process of a group
  by doing disable replication when he/she needs to make adjustments to
  the group.

* A tenant wants the ability to verify that the data will not be compromised
  after a failover before a true DR strikes. He/she is only concerned about
  his/her own application and wants to be able to test the failover of
  his/her own group of volumes without affecting the entire backend.

* A tenant wants to decide when and whether to failover when the disaster
  happens.

Proposed change
===============

The Generic Volume Group design is in a separate spec [3] and this spec
depends on it. The URL for the group action API is from the Generic Volume
Group spec:

    '/v3/<tenant_id>/groups/<group_uuid>/action'

The following actions will be added to support replication for a group of
volumes.

* enable_replication

* disable_replication

* failover_replication

* list_replication_targets

If a volume is "in-use", a "allow-attached-volume" flag needs to set to
True for failover_replication to work. This is to prevent the tenant
from abusing the failover API and causing problems when forcefully
failing over an attached volume. Either the tenant or the admin should
detach the volume before failing over, or it will be handled by a higher
layer project such as Karbor.

Group type needs to have the following group_spec to support
replication::

    'group_replication_enabled': <is> True
    or
    'consistent_group_replication_enabled': <is> True

Driver also needs to report ``group_replication_enabled`` or
``consistent_group_replication_enabled`` as capabilities.

Karbor or another data protection solution can be the consumer of the
replication API in Cinder. The flow is like this:

* Admin configures cinder.conf.

* Admin sets up group type and volume types in Cinder.

* Tenant creates a group that supports replication in Cinder.

* Tenant creates volumes and puts into the group in Cinder.

* In Karbor, a tenant chooses what resources to put in a protection group.
  For Cinder volume, Karbor can detect whether a selected volume belongs
  to a replication group and automatically selects all volumes in that
  group. For example, Karbor can check whether a Cinder volume group has
  ``group_replication_enabled`` or ``consistent_group_replication_enabled``
  in its group specs.

* There are APIs to create/delete/update/list/show a group in the Generic
  Volume Group spec [3]. Karbor or another data protection solution can use
  those APIs to manage the groups for the tenant.

* In Karbor, a protection action can be executed depending on the
  protection policies. If the protection action triggers a fail over,
  Karbor can call Cinder's replication API to failover the replication
  group.

* As discussed during the Cinder Newton midcycle meetup [4], driver can
  report ``replication_status`` as part of volume_stats. The volume manager
  will check the ``replication_status`` in volume_stats and update database
  if ``replication_status`` is ``error``.

Alternatives
------------

Do not add a generic volume group construct. Instead, just use the existing
consistency group to implement group replication.

Data model impact
-----------------
None

REST API impact
---------------

Group actions will be added:

* enable_replication

* disable_replication

* failover_replication

* list_replication_targets

URL for the group action API from the Generic Volume Group design::
    '/v2/<tenant_id>/groups/<group_uuid>/action'

* Enable replication

  - Method: POST

  - JSON schema definition for V3:

  .. code-block:: python

        {
            'enable_replication': {}
        }

  - Normal response codes: 202

  - Error response codes: 400, 403, 404

* Disable replication

  - Method: POST

  - JSON schema definition for V3:

  .. code-block:: python

        {
            'disable_replication': {}
        }

  - Normal response codes: 202

  - Error response codes: 400, 403, 404

* Failover replication

  - Method: POST

  - JSON schema definition for V3:

  .. code-block:: python

        {
            'failover_replication':
            {
                'allow_attached_volume': False,
                'secondary_backend_id': 'vendor-id-1',
            }
        }

  - Normal response codes: 202

  - Error response codes: 400, 403, 404

* List replication targets

  - Method: POST

  - JSON schema definition for V3:

  .. code-block:: python

        {
            'list_replication_targets': {}
        }

  - Response example:

  .. code-block:: python

       {
           'replication_targets': [
               {
                   'backend_id': 'vendor-id-1',
                   'unique_key': 'val1',
                   ......
               },
               {
                   'backend_id': 'vendor-id-2',
                   'unique_key': 'val2',
                   ......
               }
            ]
       }

  - Response example for non-admin:

  .. code-block:: python

       {
           'replication_targets': [
               {
                   'backend_id': 'vendor-id-1'
               },
               {
                   'backend_id': 'vendor-id-2'
               }
            ]
       }

Security impact
---------------
None

Notifications impact
--------------------
Notifications will be added for enable, disable, and failover replication.

Other end user impact
---------------------

python-cinderclient needs to support the following actions.

* enable_replication

* disable_replication

* failover_replication

* list_replication_targets

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------

Driver developers need to modify drivers to support Tiramisu.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xing-yang

Other contributors:
  jgriffith

Work Items
----------

1. Change capability reporting. Driver needs to report
   ``group_replication_enabled`` or
   ``consistent_group_replication_enabled``.

2. Add group actions to support group replication.

Dependencies
============
This is built upon the Generic Volume Group spec which is already
merged and implemented.

Testing
=======

New unit tests will be added to test the changed code.

Documentation Impact
====================

Documentation changes are needed.

References
==========
[1] Replication v2.1 Cheesecake Spec:
    https://specs.openstack.org/openstack/cinder-specs/specs/mitaka/cheesecake.html

[2] Replication v2.1 Cheesecake Devref:
    https://github.com/openstack/cinder/blob/master/doc/source/devref/replication.rst

[3] Generic Volume Group:
    https://github.com/openstack/cinder-specs/blob/master/specs/newton/generic-volume-group.rst

[4] Newton Midcycle Meetup:
    https://etherpad.openstack.org/p/newton-cinder-midcycle-day3

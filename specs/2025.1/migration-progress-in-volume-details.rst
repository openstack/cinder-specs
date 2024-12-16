..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Show migration_progress in volume details
=========================================

https://blueprints.launchpad.net/cinder/+spec/migration-progress-in-volume-details

This spec proposes adding a ``migration_progress`` field to the volume
details API response when a volume is migrating.

Problem description
===================

Volume migration might be facilitated by the backend driver, but often
requires cinder or nova to copy data from the source to the new volume.  For
large volumes this can take a long time to complete (multiple hours, even
days). Currently there's no way to track the copy operation's progress, which
makes it difficult to predict when the migration will complete.

Use Cases
---------

- As a cloud admin who is migrating a volume, I need to know it isn't taking
  so much time to complete that I fear the operation is stuck.

- As a cloud admin who is migrating a volume, I would like to track the
  migration progress in order to gauge how much longer it might take for the
  operation to complete.

Proposed change
===============

The API code for showing a volume's details would use a microversion to
optionally include a ``migration_progress`` field. The field would be included
in the API response only when the volume is migrating, and when permitted by
the microversion. As the existing ``migration_status`` field is admin-only,
the new field would also be admin-only.

The value would be obtained via an RPC that fetches a response from either the
volume service associated with the volume, or from nova when the volume is
attached and nova is copying the data. See the references for the proposed nova
feature. The RPC to the volume service or nova would be executed only when the
following criteria are met:

* The context is admin

* The volume's ``migration_status`` is "migrating"

* The API request satisfies the new microversion

The volume service handling the RPC will determine how the volume is migrating
in order to provide the proper response.

* When a migration requires cinder to copy the data then the RPC response will
  be a string representing a percent completion (e.g. "42" for 42%
  complete). This is the same response that nova returns when it provides a
  ``swap_progress`` response when its swapping a volume attachment.

* When a migration is handled by the volume's driver then the RPC response
  will be a TBD string to indicate the migration is driver-assisted. The
  subset of cinder drivers that support driver-assisted migration do not
  expose an API for reporting migration progress, and the expectation is
  drivers complete the migration fast enough to provide a satisfactory user
  experience (i.e. the use cases are covered).

* If the migration has completed by the time the RPC is received, the response
  will be ``None``.

At the API layer, the ``migration_progress`` field will be included in the
volume details response only when the RPC response is a non-empty string. For
attached volumes, nova's ``swap_progress`` will be reported as the volume's
``migration_status``.

Alternatives
------------

An earlier idea was to enhance cinder so it tried to use "driver assisted
volume migration" on attached volumes. The hope was that cinder drivers would
be capable of migrating data fast enough to alleviate concerns about the
process taking too long. This idea was rejected for multiple reasons:

* Not all cinder drivers support driver-assisted migration, and most of them
  explicitly disallow migrating attached volumes.

* Interactions between cinder and nova would be complicated. Cinder would need
  to tell nova to pause IO prior to migrating the volume, and then unpause
  after migration completes.

Data model impact
-----------------

TBD, but hopefully none. It may be necessary to add a DB field to facilitate
obtaining the proper RPC response. This will be carefully considered during
the design phase, with a goal of minimizing impact on the DB.

REST API impact
---------------

Add a new microversion to the ``GET /v3/volumes/{volume_id}`` API to include
an optional ``migration_progress`` field in the response when the
``migration_status`` field is ``migrating``, and the API context is for an
admin.

.. code-block:: json

    {
        "volume": {
            "attachments": [],
            "availability_zone": "nova",
            "bootable": "false",
            "consistencygroup_id": null,
            "created_at": "2018-11-29T06:50:07.770785",
            "description": null,
            "encrypted": false,
            "id": "f7223234-1afc-4d19-bfa3-d19deb6235ef",
            "links": [
                {
                    "href": "http://127.0.0.1:45839/v3/89afd400-b646-4bbc-b12b-c0a4d63e5bd3/volumes/f7223234-1afc-4d19-bfa3-d19deb6235ef",
                    "rel": "self"
                },
                {
                    "href": "http://127.0.0.1:45839/89afd400-b646-4bbc-b12b-c0a4d63e5bd3/volumes/f7223234-1afc-4d19-bfa3-d19deb6235ef",
                    "rel": "bookmark"
                }
            ],
            "metadata": {},
            "migration_status": "migrating",
            "multiattach": false,
            "name": null,
            "os-vol-host-attr:host": null,
            "os-vol-mig-status-attr:migstat": null,
            "os-vol-mig-status-attr:name_id": null,
            "os-vol-tenant-attr:tenant_id": "89afd400-b646-4bbc-b12b-c0a4d63e5bd3",
            "replication_status": null,
            "size": 10,
            "snapshot_id": null,
            "source_volid": null,
            "status": "creating",
            "updated_at": null,
            "user_id": "c853ca26-e8ea-4797-8a52-ee124a013d0e",
            "volume_type": "__DEFAULT__",
            "provider_id": null,
            "group_id": null,
            "service_uuid": null,
            "shared_targets": true,
            "cluster_name": null,
            "volume_type_id": "5fed9d7c-401d-46e2-8e80-f30c70cb7e1d",
            "consumes_quota": true,
            "migration_progress": "42"
       }
    }

Security impact
---------------

None. The ``migration_progress`` will be admin-only.


Active/Active HA impact
-----------------------

None, I presume.

Notifications impact
--------------------

None.

Other end user impact
---------------------

Both the openstack and cinder cli will automatically show the new
``migration_progress`` field if it's present in the API response.

Performance Impact
------------------

There will be no performance impact unless the volume is migrating. When it
is migrating, an RPC is requred to fetch the migration progress data, either
from nova (when the volume is attached) or from the volume service.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Upgrade impact
--------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <abishop@redhat.com>

Work Items
----------

* Add a new microversion in cinder and bump the MAX in python-cinderclient
* Add a new field to the volume details response in a new microversion
* Add an RPC to fetch the migration progress from the volume service
* Update the API code to invoke the appropriate (cinder or nova) RPC
* Extend existing unit and functional tests
* Update API documentation

Dependencies
============

Tracking the migration progress of an attached volume requires a nova feature.
See the references for the nova feature spec.

Testing
=======

TBD. The tricky part for incorporating something into a tempest test is
catching a migration operation during the time it's copying the volume data,
which is the only time when the ``migration_progress`` field will be present
in the volume details response.

Documentation Impact
====================

The Block Storage API reference documentation will need to be updated, but
there should be no impact on other user or admin facing documentation.

References
==========

* https://review.opendev.org/c/openstack/nova-specs/+/937485

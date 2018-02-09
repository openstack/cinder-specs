..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Support availability zone in volume type
========================================

https://blueprints.launchpad.net/cinder/+spec/support-az-in-volumetype

This blueprint proposes to support setting a volume type's availability zones.

Problem description
===================

In a newly deployed region environment, the volume type (SSD, HDD or others)
may only exist on part of the AZs, but end users have no idea which AZ is
allowed for one specific volume type and they can't realize that only when
the volume failed to be scheduled to backend.

Use Cases
=========

Administrators can update the available AZs for volume type and UI or end
users can know valid volume types in current AZ.

Proposed change
===============

As the discussion result during Rocky PTG `[1]`_, we would propose to use
volume type's ``extra specs`` to support this. Since ``extra specs`` is
designed for generic use, we would propose to introduce a reserved key
``os-extended:availability_zones`` for extra specs. Administrator can
create/update this attribute in the format as below to support AZ in
volume type::

    {
    "volume_type": {
        "id": "6685584b-1eac-4da6-b5c3-555430cf68ff",
        "name": "az_type",
        "description": "availability zone type 001",
        "is_public": true,
        "extra_specs": {
            "os-extended:availability_zones": "az1,az2,az3"
            }
        }
    }

**NOTE**: Our capability filter will skip any extra spec which has
key in the format of "prefix:key_name".
Since this use case is more related to UI or other endpoint concerns, we
would also propose to support filter volume type via extra specs as well::

    Request example:
    /v3/{project_id}/types?extra_specs={'os-extended:availability_zones':'az1'}

Also, it's mentionable that the extra spec availability zones will be
handled in a slightly different way when compared with the rest during
filtering. Cinder will always try inexact match for this specific value,
for instance, when extra spec ``os-extended:availability_zones`` is set
as below, both ``az1`` and ``az2`` are valid input for query::

    "os-extended:availability_zones": "az1,az2,az3"


In order to fully support type's AZ in Cinder, there are also some changes
on Cinder framework.

1. Now the intersection of AZs between volume type's and other inputs (user
   input, source resource or default configuration) will be calculated when
   creating a new volume and 400 Bad Request will be raised if this
   intersection is empty.
2. It's possible now we could pass an AZ list to the scheduler when creating
   or retyping, workflow and ``AvailabilityZoneFilter`` will be updated
   to support this.

**NOTE**: Cinder will raise 400 Bad Request as well if type's
availability_zones attribute is set but value is empty.


Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

1. Now Cinder supports filter volume type with extra specs::

    GET: /V3/{tenant_id}/types?extra_specs={'spec_name':'spec_value'}

    Response BODY:
    {
        "volume_types": [{
            "name": "volume_type1",
            "description": "volume_type1",
            "extra_specs": {"spec_name": "spec_value"},
        }]
    }

2. API version bump will be required for these changes.

Cinder-client impact
--------------------

Volume type's CLIs will be updated corresponding to the API change
at server side.


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

There would be a slight performance impact when filtering volume types
with extra specs.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Support availability zone and extra spec filtering in volume type API.
* Update cinder workflow to support availability zone lists.
* Add related unit testcases.
* Update cinderclient to support filter volume type with extra specs.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

* Update the API reference.
* Add administrator documentation to advertise the reserved key
  ``os-extended:availability_zones`` and explain why&when&how
  should this can be used.

References
==========

_`[1]`: https://etherpad.openstack.org/p/cinder-ptg-rocky-wednesday

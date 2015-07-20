..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Ability to update volume type public status
===========================================

https://blueprints.launchpad.net/cinder/+spec/volume-types-public-update

This proposal is to add the ability to update volume type is_public status.

Problem description
===================

Currently, the v2 volume type update api doesn't support updating volume type's
public status. All volume types created are public by default if not specified,
It is not possible to update a existing volume_type is_public status.
It is necessary to add updating public status for volume type. If a volume
type updated from public to private, the volumes created with this type will
not be affected, but the user without access will not be able to create volume
with this type anymore.

Use Cases
=========

Suppose the admin created a volume type. And he/she wants to make the volume
type not public and add access to specified projects.

Proposed change
===============

* Modify volume_type update API adding is_public property support.

Alternatives
------------


Data model impact
-----------------


REST API impact
---------------

Volume type update change

* Update volume type API
  * V2/<tenant id>/types/volume_type_id
  * Method: PUT
  * JSON schema definition for V2::

        {
            "volume_type":
            {
                "name": "test_type",
                "description": "Test volume type",
                "is_public": "False" # new
            }
        }

  * In the existing update volume type API, add a new parameter "is_public" to
    allow updating public status for volume type.

Security impact
---------------


Notifications impact
--------------------


Other end user impact
---------------------

python-cinderclient needs to be changed to support the modified API.

* Update volume type
  cinder type-update --name <name> --description <description>
  --is-public <is-public>

Performance Impact
------------------


Other deployer impact
---------------------


Developer impact
----------------


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  liyingjun

Other contributors:

Work Items
----------

1. API change:
   * Modify Update Volume Type API.

Dependencies
============

Testing
=======

New unit tests will be added to test the changed code.

Documentation Impact
====================

Documentation changes are needed.

References
==========

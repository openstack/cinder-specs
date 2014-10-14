..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Volume Type Description
==========================================

https://blueprints.launchpad.net/cinder/+spec/volume-type-description

This blueprint proposes to add the description information for a volume type
and allow user to find out what is the default volume type.
Volume type description and default volume type can help a user make an
informed decision when the user creates a volume.


Problem description
===================

* Cinder volume types only have abstract names like gold, silver and bronze
  or whatever the Openstack administrator comes up with. When a user creates
  a volume and chooses which volume type to use, currently it is difficult
  for a user to find out what the volume type means.

* When a user creates a volume without specifying a volume type, once the
  volume gets created, the volume type assigned to that volume could be a
  default volume type or None. There is no way for a user to find out
  what is the default volume type.


Proposed change
===============

* When an administrator creates a volume type, allows him/her to enter some
  descriptions for the volume type. The length of the description should be
  between 0 - 255. It should be in the form of a string.

* Administrator can also update the descriptions of an existing volume type.

* User should be able to get all the descriptions of all the volume
  types.

* User should be able to get the descriptions of a volume type.

* User should be able to find out what is the default volume type.

Alternatives
------------

None

Data model impact
-----------------

Database schema changes:

* A new description column will be added to the volume_types table.

::

  mysql> DESC volume_types;
  +--------------+--------------+------+-----+---------+-------+
  | Field        | Type         | Null | Key | Default | Extra |
  +--------------+--------------+------+-----+---------+-------+
  | created_at   | datetime     | YES  |     | NULL    |       |
  | updated_at   | datetime     | YES  |     | NULL    |       |
  | deleted_at   | datetime     | YES  |     | NULL    |       |
  | deleted      | tinyint(1)   | YES  |     | NULL    |       |
  | id           | varchar(36)  | NO   | PRI | NULL    |       |
  | name         | varchar(255) | YES  |     | NULL    |       |
  | qos_specs_id | varchar(36)  | YES  | MUL | NULL    |       |
  | description  | varchar(255) | YES  |     | NULL    |       |
  +--------------+--------------+------+-----+---------+-------+


Database data migration:

* Existing volume types will have description empty.

REST API impact
---------------

* Update "create volume type" REST request to to include description field.

* Update "list volume types" REST response to include description field for
  each volume type

* Update "show volume type information" REST response to include description
  field.

* Add "update volume type" REST API to update an existing volume type.
  * PUT /v2/{tenant_id}/types/{volume_type_id}

  * JSON request schema definition::

        'volume_type': {
            'name': 'gold',
            'description': 'gold means very important'
        }

  * JSON response schema definition::

        'volume_type': {
            'id': '4b502fcb-1f26-45f8-9fe5-3b9a0a52eaf2',
            'name': 'gold',
            'description': 'gold means very important',
            'extra_specs':{}
        }

  * Normal http response code: 200

  * Expected error http response code: 400, 404, 500, 409

* Add "get default volume type" REST API

 * GET /v2/{tenant_id}/types/default

 * JSON response schema definition::

        'volume_type': {
            'id': '4b502fcb-1f26-45f8-9fe5-3b9a0a52eaf2',
            'name': 'bronze',
            'description': 'bronze means limited storage',
            'extra_specs':{}
        }

 * Normal http response code: 200

 * Expected error http response code: 404

Security impact
---------------

Creating or updating volume type description is usually only allowed for an
administrator user.

Regular user should be able to get a volume type, list volume types and get
the default volume type.

The APIs accesses will be controlled by security policy.

Notifications impact
--------------------

Volume type creation has already sent a notification when creation ends or
has errors. Will send a notification when updating a volume type ends or has
errors.

Other end user impact
---------------------

* python-cinderclient will be changed to reflect the API changes.

  volume_types.create will be updated to include description.
  volume_types.get_default will be added to show the default volume type.
  volume_types.update will be added to update the description of the volume
  type.

* Horizon will have corresponding UI changes to deal with the descriptions for
  a volume type after python-cinderclient implementation.

Performance Impact
------------------

Adding another db column in the volume type field means that each fetch of a
volume type will pull the description. It is a minimal impact for individual
volume type reads. It is doubtful if there will be a significant number of
volume types created with lots of descriptions. So, the performance impact
should be minimal.

Other deployer impact
---------------------

DB volume_types table migration and associated volume service restart will
require orchestration and a short service downtime. Transient API errors might
happen between the schema migration and the deployment of the new code
(which ever order they are done in).

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gloria-gu

Other contributors:
  None

Work Items
----------

* Implement Cinder API changes.

* Implement DB schema changes.

* Implement DB migration script changes.

    cinder/db/sqlalchemy/migrate_repo/versions

* Implement python-cinderclient changes.

* Cinder API unit Tests.

* DB migration unit test changes.

    cinder/tests/test_migrations.py


Dependencies
============

Horizon blueprint will depend on this spec:

* https://blueprints.launchpad.net/horizon/+spec/volume-type-description


Testing
=======

* Update the unit tests to reflect the API changes.
* Update the DB migration tests.


Documentation Impact
====================

* The Cinder API documentation will need to be updated to reflect the API
  changes.
* The Cinder client documentation will be need to be updated to reflect the
  changes.


References
==========

None

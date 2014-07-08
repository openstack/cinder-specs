..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Private Volume Types
====================

https://blueprints.launchpad.net/cinder/+spec/private-volume-types

Cinder volume types are visible to all users, regardless of their project.

This blueprint suggests the introduction of private volume types.


Problem description
===================

Some volume types should only be restricted. Examples are test volume types
where a new technology is being tried out or ultra high performance volumes
for special needs where most users should not be able to select these volumes.


Proposed change
===============

Similar approaches are taken with the is_public flag on flavors in Nova.
We should leverage the work done in Nova and port it for Cinder volume types.

Volume types currently do not have an owner associated to them. This feature
does not suggest the introduction of an owner for various reasons, one being
that it is impossible to find the original owner of an existing volume type.

The proposed approach is the one already in place in Nova:

* Volume types are public by default
* Private volume types can be created by setting the is_public boolean field
  to False at creation time.
* Access to a private volume type can be controlled by adding or removing
  a project from it.
* Private volume types without projects are only visible by users
  with the admin role/context.

Alternatives
------------

There is no known alternative ways to restrict access to a volume type.

Data model impact
-----------------

Database schema changes:

* A new is_public boolean column will be added to the volume_types table.
* A new volume_type_projects table will be created for projects having access
  to a particular volume types. There will be one entry in volume_type_projects
  table for every volume_type_id and project_id combination.
  It will be a many-to-many relationship.

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
  | is_public    | tinyint(1)   | YES  |     | NULL    |       |
  +--------------+--------------+------+-----+---------+-------+
  8 rows in set (0.00 sec)

  mysql> DESC volume_type_projects;
  +----------------+--------------+------+-----+---------+----------------+
  | Field          | Type         | Null | Key | Default | Extra          |
  +----------------+--------------+------+-----+---------+----------------+
  | id             | int(11)      | NO   | PRI | NULL    | auto_increment |
  | created_at     | datetime     | YES  |     | NULL    |                |
  | updated_at     | datetime     | YES  |     | NULL    |                |
  | deleted_at     | datetime     | YES  |     | NULL    |                |
  | volume_type_id | varchar(36)  | YES  | MUL | NULL    |                |
  | project_id     | varchar(255) | YES  |     | NULL    |                |
  | deleted        | tinyint(1)   | YES  |     | NULL    |                |
  +----------------+--------------+------+-----+---------+----------------+
  7 rows in set (0.00 sec)

Database data migration:

* Existing volume types will be marked as public (is_public=1)

REST API impact
---------------

* Extend volume type creation response to include is_public field
* Extend volume type list to include is_public field
* Extend volume type detail to include is_public field
* Add ability to list projects having access to a specific volume type
* Add ability to add/remove access for a project to a specific volume type
* Add policy for the new extension

Security impact
---------------

This change introduces the concept of private volume types.

Notifications impact
--------------------

None

Other end user impact
---------------------

* Horizon should be updated to support this new extension.
* python-cinderclient should be updated to allow the use of this new extension.

Proposed python-cinderclient shell interface::

type-access-add --volume-type <type> --project-id <project_id>
    Add type access for the given project.

type-access-list --volume-type <type>
    Print access information about the given type.

type-access-remove --volume-type <type> --project-id <project_id>
    Remove type access for the given project.


Performance Impact
------------------

The extension adds an is_public field to all returned volumes.

Special care should be taken to not generate N requests per volume list.
This can easily be addressed by a caching mechanism at the API layer.

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
  mgagne

Other contributors:
  None

Work Items
----------

* Implement os-volume-type-access Cinder extension
* Add support for os-volume-type-access extension to python-cinderclient
* Add support for os-volume-type-access extension to Horizon


Dependencies
============

None


Testing
=======

* Unit tests already in place in Nova for flavors will be ported
  for Cinder volume types.
* Use cases should be added to Tempest.


Documentation Impact
====================

* Need to document the new os-volume-type-access Cinder extension.


References
==========

* http://lists.openstack.org/pipermail/openstack-operators/2014-June/004561.html

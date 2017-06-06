..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Support metadata for backup
===========================

https://blueprints.launchpad.net/cinder/+spec/metadata-for-backup

Add a new "metadata" property for backup resource.

Problem description
===================

Backup resource lost the ability for getting/setting metadata property.


Use Cases
=========

The metadata here for backup is the descriptive metadata. It's used for
discovery and identification. Users could add key-value pairs for the backups
to describe them. And users can also filter backups with specified metadata.


Proposed change
===============

1. The "metadata" property will be added to backup object.

2. A new DB table "backup_metadata" will be created.
::

    -----------------------------------------------------------------------
    | created_at | updated_at | deleted_at | id | backup_id | key | value |
    -----------------------------------------------------------------------
    |            |            |            |    |           |     |       |
    -----------------------------------------------------------------------

The primary key is "id".

3. The backup create/update API will be updated to support "metadata".
::

    POST /v3/{project_id}/backups
    PUT  /v3/{project_id}/backups/{backup_id}

    the request body can contain "metadata".
    {
        "metadata":{
            "key1": "value1",
            "key2": "value2"
        }
    }

4. A set of new APIs will be created. It's used for backup metadata's CRUD.
::

    GET    /v3/{project_id}/backups/{backup_id}/metadata
    show a backup's metadata

    POST   /v3/{project_id}/backups/{backup_id}/metadata
    create or replaces metadata for a backup

    PUT    /v3/{project_id}/backups/{backup_id}/metadata
    replace all the backup's metadata

    GET    /v3/{project_id}/backups/{backup_id}/metadata/{key}
    show a backup's metadata for a specific key

    DELETE /v3/{project_id}/backups/{backup_id}/metadata/{key}
    delete a specified metadata

    PUT    /v3/{project_id}/backups/{backup_id}/metadata/{key}
    update a specified metadata

Alternatives
------------

Leave as it is.

Data model impact
-----------------

Backup model will be updated with new property "metadata".

REST API impact
---------------

* The backup create/update API's request body will be updated.
* A set of new APIs related to backup metadata will be created.

Security impact
---------------

None

Notifications impact
--------------------

The new APIs will send new notifications as well.

Other end user impact
---------------------

None

Performance Impact
------------------

A new "backup_metadata" table will be created so that the DB conjunction action
may let the search performance reduce a little.

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
  wangxiyuan(wxy)

Work Items
----------

* Add metadata property to backup object and bump its version.
* Create a new DB table "backup_metadata" and add db upgrade script.
* Update backup create/update API.
* Add a tuple of new APIs for backup metadata.


Dependencies
============

None


Testing
=======

* Unit tests


Documentation Impact
====================

* Api-ref need update.


References
==========

None

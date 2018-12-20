..
   This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Add user_id attribute to backup response
========================================
https://blueprints.launchpad.net/cinder/+spec/add-user-id-attribute-to-backup-response

This blueprint proposes to add ``user_id`` attribute to
the response body of list backup with detail and show backup detail APIs.

Problem description
===================

Currently, there is a ``user_id`` field in the ``backups`` table. These
fields are very useful for admin to manage the backup file, but this
is not returned in response body. So it is difficult to manage the resources
under the project. If there are multiple users under one project, it is
impossible to distinguish which user the backup file belongs to.

Use Cases
=========

In large scale environment, lots of backups resources were created in system,
that we can only see the project to which the backup file belongs, but we
cannot know to which user the backup belongs.

Administrators would like the ability to identify the users that have created
backups.

Proposed change
===============

This spec proposes to add ``user_id`` attribute to the
response body of list backup with detail and show backup detail APIs.

Add a new microverion API to add ``user_id`` attribute
to the response body of list backup with detail and show backup detail APIs:

- List backups with detail GET /v3/{project_id}/backups/detail

- Show backup detail GET /v3/{project_id}/backups/{backup_id}

Alternatives
------------

The admin/user could get ``user_id`` from the context as a log print, but
it's difficult to find it out easily, especially when the user wants to find
a very old backup file.

REST API impact
---------------

Add a new microversion in Cinder API.

List backups with detail::

  GET /v3/{project_id}/backups/detail
  Response BODY:
  {
      "backups": [{
          ...
          "user_id": "515ba0dd59f84f25a6a084a45d8d93b2"
      }]
  }

Show backup detail::

  GET /v3/{project_id}/backups/{backup_id}
  Response BODY:
  {
      "backups": [{
          ...
          "user_id": "515ba0dd59f84f25a6a084a45d8d93b2"
      }]
  }

Calling this method shows a ``user_id`` for volume backup.
It is intended for admins to use, which is used to display the user to which
the backup file belongs, and controlled by ``BACKUP_ATTRIBUTES_POLICY``.

Data model impact
-----------------

None

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

None

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
  Brin Zhang <zhangbailin@inspur.com>

Work Items
----------

* Add a new microversion.
* Add ``user_id`` to the response body of list backup
  with detail and show backup detail APIs.
* Add the related unit tests.
* Update related list backup with detail and show detail api doc.

Dependencies
============

None

Testing
=======

* Unit-tests, tempest and other related test should be implemented

Documentation Impact
====================

None

References
==========

None

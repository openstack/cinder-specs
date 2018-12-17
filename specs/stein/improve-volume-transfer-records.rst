..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Improve volume transfer records
===============================
https://blueprints.launchpad.net/cinder/+spec/improve-volume-transfer-records

This blueprint proposes to improve volume transfer records by adding
``source_project_id`` and ``destination_project_id``, ``accepted`` fields to
``transfer`` table and related api responses, makes it easier for users to
trace the volume transfer history.

Problem description
===================

Currently, the volume transfer record does not include the destination
project_id after transferring and the source project_id before transferring.
These fields are very useful for admins and operators to trace the transfer
history.

And also once the transfer is deleted, the user can't determine if this
transfer had been accepted or not before it was deleted.

It is bad for admins and users to trace volume historical track between
project and audit the volume records.

Use Cases
===================
* In order to trace the volume transfer history, the admin wants to know who
  was the volume owner before transferring.
* The admin wants to know whether a deleted transfer had been accepted or not.

Proposed change
===============
This spec proposes to do

1. Add three new fields to ``transfer`` table:

   * ``source_project_id``, this field records the source project_id
     before volume transferring.

   * ``destination_project_id``, this field records the destination project_id
     after volume transferring.

   * ``accepted``, this field records if this transfer was accepted or not.

2. Add a new microverion API to add above fields to the response of follow
   API:

   - Create a volume transfer POST /v3/{project_id}/volume-transfers

   - Show volume transfer detail GET /v3/{project_id}/volume-transfers

   - List volume transfer and detail GET
     /v3/{project_id}/volume-transfers/detail

   And the response of "List volume transfer (non-detail)" API will not
   include these fields.

Alternatives
------------

The admins could find part of volume transferring info from log, but it's
difficult to find it out easily, especially when the user wants to audit a
very old volume transfer.


REST API impact
---------------

A new microversion will be created to add these new added fields to transfer
related API responses.

Data model impact
-----------------

None

Security impact
---------------

None

Notifications impact
--------------------

Notifications will be changed to add these new added fields.

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
  Yikun Jiang <yikunkero@gmail.com>

Work Items
----------
* Add ``source_project_id`` and ``destination_project_id``, ``accepted``
  fields to ``transfer`` table
* Add ``source_project_id`` and ``destination_project_id``, ``accepted``
  fields to related API.
* Implement changes for python-cinderclient to support list transfer with
  ``--detail``.
* Update related transfer api doc.

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

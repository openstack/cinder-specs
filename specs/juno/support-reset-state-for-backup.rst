..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Backup State Reset API
======================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/support-reset-state-for-backup

Provide an API to reset the state of a backup being stucked in creating or
restoring.

Problem description
===================

Since there are volume reset-state function and snapshot reset-state function,
backup also needs reset-state as well.
When creating or restoring backup, it may leave backup stucked in creating or
restoring status due to database down or rabbitmq down, etc..
Currently we could only solve these problems by restarting cinder-backup. This
bp is to another mean for administrators to solve these problems by calling
backup reset state API.

1. Resetting status from creating/restoring to available

 1) restoring --> available
    Directly change the backup status to 'error', because the backup data is
    already existed in storage backend.
 2) creating --> available
    Use backup-create routine as an example to illustrate what benefit we can
    get from backup-reset function. Backup-create routine first backup volume
    and metadatas, and then update the status of volume and backup. If database
    just went down after update the volume's status to 'available', leaving the
    backup's status to be 'creating' without having methods to deal with
    through API.

 If we have reset-state API and resetting status from creating to available, we
 first verify whether the backup is ok on storage backend.
 If so, we change backup status from creating to available.
 If not, we throw an exception and change backup status from creating to error.

2. Resetting status from creating/restoring to error
Directly change the backup status to 'error' without restart cinder-backup.

Use Cases
=========

Proposed change
===============

A new API function and corresponding cinder command will be added to reset
the status of backups.

The initial proposal is to provide a method for administrator to handle the
backup item stucked in status like creating or restoring.

Alternatives
------------

Login in the cinder database, use the following update sql to change the
backup item.

::

    update backups set status='some status' where id='xxx-xxx-xxx-xxx';

Data model impact
-----------------
None

REST API impact
---------------

Add a new REST API to reset backup states:
  * POST /v2/{tenant_id}/backups/{id}/action

JSON request schema definition::

    'backup-reset_status': {
        'status': 'available'
    }

Normal http response code:
    202

Expected error http response code:
    400

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------

A new command, backup-reset-state, will be added to python-cinderclient. This
command mirrors the underlying API function.

Resetting the status of a backup can be performed by:
$ cinder backup-reset-state --state <state> <backup>


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
  ling-yun

Work Items
----------

* Implement REST API
* Implement cinder client functions
* Implement cinder command

Dependencies
============
None

Testing
=======
None


Documentation Impact
====================

The cinder client documentation will need to be updated to reflect the new
command.

The cinder API documentation will need to be updated to reflect the REST API
changes.


References
==========

None

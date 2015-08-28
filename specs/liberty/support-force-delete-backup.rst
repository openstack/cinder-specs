..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Backup Force Delete API
=======================

https://blueprints.launchpad.net/cinder/+spec/support-force-delete-backup

Provide an API to force delete a backup being stucked in creating or
restoring, etc..

Problem description
===================

Currently there are volume force-delete and snapshot force-delete functions,
but there is not a force-delete function for backups. Force-delete for backups
would be beneficial when backup-create fails and the backup status is stuck in
'creating'. This situation occurs when database just went down after backup
volume and metadata and update the volume's status to 'available', leaving the
backup's status to be 'creating' without having methods to deal with through
API, because backup-delete api could only delete backup item in status of
'available' and 'error'.

Use Cases
=========

If backup create successfully in object storage, but become stuck in update
backup's status because database just went down. Then use force-delete API,
we could directly delete the backup item(include all the stuff in storage
backend and db entry info) without manually change the backup's status in
db to error or restart cinder-backup and call backup-delete function,
which is very useful for administrators.

Proposed change
===============

A new API function and corresponding cinder command will be added to force
delete backups.

The proposal is to provide a method for administrator to quickly delete the
backup item that is not in the status of 'available' or 'error'.

* It's an admin-only operation.

Alternatives
------------

First, login in the cinder database, use the following update sql to change
the backup item status to 'available' or 'error'.

update backups set status='available'(or 'error') where id='xxx-xxx-xxx-xxx';

Second, call backup delete api to delete the backup item.

Data model impact
-----------------
None

REST API impact
---------------

Add a new REST API to delete backup in v2::
*POST /v2/{tenant_id}/backups/{id}/action

    {
	    "os-force_delete": {}
    }

Normal http response code:
    202

Expected error http response code:
    404

Security impact
---------------
None

Notifications impact
--------------------
Delete notification should include whether force was used or not

Other end user impact
---------------------

A new command, backup-force-delete, will be added to python-cinderclient. This
command mirrors the underlying API function.

Force delete a backup item can be performed by:
$ cinder backup-force-delete <backup>


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
Need to test the force delete with an in-progress backup and ensure that it
deletes successfully and cleans up correctly.


Documentation Impact
====================

The cinder client documentation will need to be updated to reflect the new
command.

http://docs.openstack.org/admin-guide-cloud/content/managing-volumes.html

The cinder API documentation will need to be updated to reflect the REST API
changes.


References
==========

None

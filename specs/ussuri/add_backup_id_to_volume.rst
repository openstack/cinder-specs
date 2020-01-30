..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Add backup id to volume's metadata
==================================

https://blueprints.launchpad.net/cinder/+spec/add-volume-backup-id

Problem description
===================
Currently, the end user can see the source (source by snapshot or image etc.)
from the new volume created. But when the end user has
restored a backup volume, in the volume's response,
we cannot see its source. This can cause a lot of
confusion for end users.

Use Cases
=========
As an end user, I would like to know the restored volume comes
from which backup file.

Proposed change
===============
* Add the property ``src_backup_id`` to the volume's metadata,
  to record where the new volume was created from.
  When restoring from a chain of incremental backups, src_backup_id
  is set to the last incremental backup used for the restore.

Once added to the volume metadata, the ``src_backup_id`` will appear on
any API response that displays volume metadata:

* the volume-show response (GET /v3/{project_id}/volumes/{volume_id})
* the volume-list-detail response (GET /v3/{project_id}/volumes/detail)
* the volume-metadata-show response
  (GET /v3/{project_id}/volumes/{volume_id}/metadata)
* the volume-metadata-show-key response
  (GET /v3/{project_id}/volumes/{volume_id}/metadata/{key})

Vendor-specific changes
-----------------------
None

Alternatives
------------
None

Data model impact
-----------------
None

REST API impact
---------------
None.
The volume-show, volume-list-detail, and volume-metadata-show
responses are currently defined to contain a ``metadata`` element that
is either JSON null or a JSON object consisting of a list of key/value pairs.
The ``src_backup_id`` will appear in this list for appropriate volumes,
but this respects the current API and does not require a new microversion.

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
Xuan Yandong

Work Items
----------
* Add ``src_backup_id`` to the ``volume`` metadata

Dependencies
============
None

Testing
=======
* Add related unit tests
* Add related functional test
* Add tempest tests

Documentation Impact
====================
Release note should point out that since this is stored in
the volume metadata, it can be modified or removed by end
users, so operators should not rely upon it being present
for administrative or auditing purposes.

Add a similar note somewhere in the admin docs, probably a
page about volume metadata written by Cinder (which may not
currently exist). In addition to reminding admins that end users can overwrite
volume metadata, should explain how to read the 'src_backup_id'(particularly
the part about what id is used when restoring from incremental backups).

References
==========
None

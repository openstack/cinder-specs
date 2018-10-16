..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Object deletion from DB without driver interaction
==================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-delete-from-db


Problem description
===================

It's impossible to delete a volume, a backup or a snapshot when the service is
not available. There's no way to bypass driver interaction.

Use Cases
=========

Sometimes, OpenStack admins are pulling the plugs off some storage backends to
use a new one. They realize later that they need to cleanup the various
objects.


Proposed change
===============

With cinder-manage, we will have a --db-only switch under the volume delete
command. The snapshots are deleted in cascade. We will also implement backup
subcommand that acts like the volume subcommand.

Alternatives
------------

The only known workaround is to manually update multiple tables and set the
status to 'deleted' to various objects.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

These commands will be available to operators:

.. code-block:: console

 cinder-manage volume delete [--db-only] <volume-id>
 cinder-manage backup delete [--db-only] <backup-id>

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

Primary assignee: David Vallee Delisle <dvd@redhat.com>

Work Items
----------

The following changes will be done under cinder-manage command:
* Add a --db-only switch to the volume delete command.
* Add a backup delete subcommand with --db-only support.
* The snapshots are deleted in cascade when a volume is deleted.

The following changes will be done in the rpcapis:
* Add a db_only argument in the delete function

The following changes will be done in the manages:
* Add a db_only argument in the delete function

Dependencies
============

None

Testing
=======

Create tests to validate the delete volume and the delete backup are really
deleting.

Documentation Impact
====================

Man page for cinder-manage and any associated documentation will be updated.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
cinder db purge utility
==========================================

https://blueprints.launchpad.net/cinder/+spec/database-purge

This spec adds the ability to sanely and safely purge deleted rows from
the cinder database for all relavent tables. Presently, we keep all deleted
rows, or archive them to a 'shadow' table. I believe this is unmaintainable
as we move towards more upgradable releases. Today, most users depend on
manual DB queries to delete this data, but this opens up to human errors.

The goal is to have this be an extension to the `cinder-manage db` command.
Similar specs are being submitted to all the various projects that touch
a database.

Problem description
===================

Very long lived Openstack installations will carry around database rows
for years and years. To date, there is no "mechanism" to programmatically
purge the deleted data. The archive rows feature doesn't solve this.

Use Cases
=========

Operators should have the ability to purge deleted rows, possibily on a
schedule (cronjob) or as needed (Before an upgrade, prior to maintenance)
The intended use would be to specify a number of days prior to today for
deletion, e.g. "cinder-manage db purge 60" would purge deleted rows that
have the "deleted_at" column greater than 60 days ago

Project Priority
-----------------

Low

Proposed change
===============

The proposal is to add a "purge" method to DbCommands in
cinder/cinder/cmd/manage.py
This will take a number of days argument, and use that for a data_sub match
Like:
delete from instances where deleted != 0 and deleted_at > data_sub(NOW()...)

Alternatives
------------

Today, this can be accomplished manually with SQL commands, or via script.
There is also the archive_deleted_rows method. However, this won't satisfy
certain data destruction policies that may exist at some companies.

Data model impact
-----------------

None, all tables presently include a "deleted_at" column.

REST API impact
---------------

None, this would be run from cinder-manage

Security impact
---------------

Low, This only touches already deleted rows.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

This has the potential to improve performance for very large databases.
Very long-lived installations can suffer from inefficient operations on
large tables.

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

primary author and contact.
    Abel Lopez <al592b>

Primary assignee:
  <al592b>

Other contributors:
  <al592b>

Work Items
----------

Add purge functionality to manage.py db/api.py db/sqlalchemy/api.py
Add tests to confirm functionality
Add documentation of feature

Dependencies
============

None

Testing
=======

The test will be written as such. Three rows will be inserted into a test db.
Two will be "deleted=1", one will be "deleted=0"
One of the deleted rows will have "deleted_at" be NOW(), the other will be
"deleted_at" a few days ago, lets say 10. The test will call the new
function with the argument of "7", to verify that only the row that was
deleted at 10 days ago will be purged. The two other rows should remain.

Documentation Impact
====================

will need to add documentation of this feature

References
==========

This was discussed on both the openstack-operators mailing list and the
openstack-developers mailing lists with positive feedback from the group.

http://lists.openstack.org/pipermail/openstack-dev/2014-October/049616.html

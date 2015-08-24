..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Cinder DB Archiving
===================

https://blueprints.launchpad.net/cinder/+spec/db-archiving

We actually don't delete rows from db, we mark it only as deleted
(we have special column). So the DB is growing and growing,
so this causes problem with performance.


Problem description
===================

A lot of unused data in the DB causes different kind of problems with
performance and maintaining:

* Need to filter 'marked as deleted' rows in every query

* DB contains a lot of unused data in each table and operators
  need to filter it if they use database directly in an emergency case

* Storage usage utilization (low priority)

Use Cases
=========

Proposed change
===============

Create shadow table for each table and copy "deleted" rows from main to shadow
table.

We need to provide several method for archiving data:

* archive not more than N "deleted" rows in each table

* archive "deleted" rows older than specified data

* archive all "deleted" data

Shadowing could be started as a periodic task or admin management util.

Alternatives
------------

Create event-based solution to make opportunity operators to subscribe on
"deleting" events and store deleted data somewhere.

Data model impact
-----------------

* Create shadow tables

* Implement migrations to store current "deleted" rows in shadow table

* Shadow tables could have blob field to store some "deleted" data and to not
  impose restrictions on database schema changes.

REST API impact
---------------

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

No performance hit from filtering out a potentially massive number of deleted
records on every query.


Other deployer impact
---------------------

* Operator or deployer could use periodic task or Cinder management tool to
  archive "deleted" data.


Developer impact
----------------

Developers should care about migrations for shadow tables as well, as for
original tables:

* table creation or deletion requires creating or deleting corresponding
  shadow tables

* when a table is modified, the shadow tables have to get modified

* when one or more columns are moved to a new table, columns from shadow table
  should also moved to a new shadow table with data migration

* downgrades should be implemented for shadow tables too: new tables
  have to get removed and the migrated columns will have to get reverted


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Ivan Kolodyazhny (e0ne)

Other contributors:
  Boris Pavlovich (boris-42)

Work Items
----------

None


Dependencies
============

None


Testing
=======

Unit tests for both API and Tempest will be implemented,


Documentation Impact
====================

Cloud Administration Guide will be updated to introduce new cinder-manage
command:

* http://docs.openstack.org/admin-guide/blockstorage-manage-volumes.html


References
==========

* Nova's spec for db archiving: https://review.openstack.org/#/c/18493/

* Discussion in openstack-dev mailing list:
http://lists.openstack.org/pipermail/openstack-dev/2014-March/029952.html

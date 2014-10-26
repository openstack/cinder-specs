..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add Cinder Support for DB2 (v10.5+)
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/db2-database

The community currently supports MySQL and PostgreSQL production databases.
Several other core projects already support DB2 (Keystone, Glance, Ceilometer,
Heat). This blueprint adds support to Cinder for DB2 as a production
database.


Problem description
===================

* Currently there is no support in the community for a deployer to run Cinder
  against a DB2 backend database.

* For anyone running applications against an existing DB2 database that wants
  to move to OpenStack, they'd have to use a different database engine to
  run Cinder in OpenStack.

* There is currently an inconsistent support matrix across the core projects
  since the majority of core projects support DB2 but Cinder does not
  yet.


Proposed change
===============

The changes required to enable Cinder to run on DB2 are limited given that
Cinder's database structure is not complicated.  No changes are currently
required to the migration scripts to enable deployment on a DB2 database.

Unit test will need to be updated to support running tests against a DB2
backend with the ibm_db_sa driver and all Cinder patches will be tested
against a Tempest full run with 3rd party CI running DB2, maintained by IBM.
Note that these 3rd Party CI runs have already been enabled for Cinder.

There is already code in Oslo's db.api layer to support common function
with DB2 like duplicate entry error handling and connection trace, so that is
not part of this spec.  Cinder, however, will need to sync up to the latest
Oslo DB code to enable DB2 support.


Alternatives
------------

Deployers can use other supported database backends like MySQL or PostgreSQL,
but this may not be an ideal option for customers already running applications
with DB2 that want to integrate with OpenStack. In addition, you could run
other core projects with multiple schemas in a single DB2 OpenStack database,
but you'd have to run Cinder separately which is a maintenance/configuration
problem.

Data model impact
-----------------

There are no impacts on the current Cinder data model.  Any issues that arise
due to future changes will be uncovered by the 3rd Party CI tests and will be
able to be addressed at that point in time.

The one exception to this statement is in the unit test path.  I made two
commits in Icehouse to start implementing the infrastructure to enable DB2
unit testing.  The commits were to add a bool_type dictionary
(2a7b11922bd9389287915c45de92ca5eed3d448e) and a time_type dictionary
(a9527de9ed3a2eae951564c3c74b7319113e8bf5).  I will need to add the
appropriate types for DB2, as was done for MySQL, as part of the unit test
changes I will be making for DB2.


REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Execution of migration unit tests on DB2 may take longer than on other
backend databases due to the time required to create the database schema.
This impact, however, will only be seen when running unit tests for DB2.

Other deployer impact
---------------------

There are no plans to migrate the Cinder database from a pre-existing
installation (I.E. MySQL) to DB2.  So deployers will need to be starting
with a fresh Cinder installation to enable DB2 as a backend database.

Developer impact
----------------

The only impact on developers is if they are adding DB API code or migrations
that don't work with DB2, they will have to adjust those appropriately, just
like we do today with MySQL and PostgreSQL. IBM ATCs would provide
support/guidance on issues like this which require specific conditions for DB2,
although for the most part, the DB2 InfoCenter provides adequate detail on how
to work with the engine and provides details on error codes.

* DB2 SQL error message explanations can be found here:
  http://pic.dhe.ibm.com/infocenter/db2luw/v10r5/index.jsp?topic=%2Fcom.ibm.db2.luw.messages.sql.doc%2Fdoc%2Frsqlmsg.html

* Information on developing with DB2 using python can be found here:
  http://pic.dhe.ibm.com/infocenter/db2luw/v10r5/index.jsp?topic=%2Fcom.ibm.swg.im.dbclient.python.doc%2Fdoc%2Fc0054366.html

* Main contacts for DB2 questions in OpenStack:

   * Matt Riedemann (mriedem@us.ibm.com) - Nova core member
   * Brant Knudson (bknudson@us.ibm.com) - Keystone core member
   * Jay Bryant (jsbryant@us.ibm.com) - Cinder core member
   * Rahul Priyadarshi (rahul.priyadarshi@in.ibm.com) - ibm_db_sa maintainer


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Email: jsbryant@us.ibm.com
  Launchpad ID: jsbryant

Work Items
----------

#. Make the test_migrations.py module work with a configured DB2 backend for
   running unit tests.

   #. Add the appropriate column types for DB2.
   #. Add support for creating the DB2 schema for Cinder testing.


Dependencies
============

* DB2 support was added to sqlalchemy-migrate 0.9 during Icehouse:
  https://blueprints.launchpad.net/sqlalchemy-migrate/+spec/add-db2-support

* There are no requirements changes in Cinder for the unit tests to work. The
  runtime requirements are the ibm-db-sa and ibm_db modules, which are both
  available from pypi. sqlalchemy-migrate optionally imports ibm-db-sa. The
  ibm-db-sa module requires a natively compiled ibm_db which has the c binding
  that talks to the DB2 ODBC/CLI driver.

* Note that only DB2 10.5+ is supported since that's what added unique index
  support over nullable columns which is how sqlalchemy-migrate handles unique
  constraints over nullable columns.


Testing
=======

* IBM is already running 3rd party CI for DB2 on Cinder.

* DB2 Unit Test for Cinder will be enabled.


Documentation Impact
====================

* The install guides in the community do not go into specifics about setting up
  the database.  The RHEL/Fedora install guide says to use the openstack-db
  script provided by openstack-utils in RDO which uses MySQL.  The other
  install guides just say that SQLite3, MySQL and PostgreSQL are widely used
  databases. So for the install guides, those generic statements about
  supported databases would be updated to add DB2 to the list. Similar generic
  statements are also made in the following places which would be updated as
  well:

   * http://docs.openstack.org/training-guides/content/developer-getting-started.html
   * http://docs.openstack.org/admin-guide-cloud/content/compute-service.html
   * http://docs.openstack.org/trunk/openstack-ops/content/cloud_controller_design.html

* There are database topics in the security guide, chapters 32-34, so there
  would be DB2 considerations there as well, specifically:

   * http://docs.openstack.org/security-guide/content/ch041_database-backend-considerations.html
   * http://docs.openstack.org/security-guide/content/ch042_database-overview.html
   * http://docs.openstack.org/security-guide/content/ch043_database-transport-security.html


References
==========

* There are Chef cookbooks on stackforge which support configuring OpenStack
  to run with an existing DB2 installation:
  http://git.openstack.org/cgit/stackforge/cookbook-openstack-common/

* A wiki document originally written to describe DB2 for OpenStack:
  https://wiki.openstack.org/wiki/DB2Enablement

* DB2 10.5 InfoCenter: http://pic.dhe.ibm.com/infocenter/db2luw/v10r5/index.jsp

* Some older manual setup instructions for DB2 with OpenStack:
  http://www.ibm.com/developerworks/cloud/library/cl-openstackdb2/index.html

* ibm-db-sa: https://code.google.com/p/ibm-db/source/clones?repo=ibm-db-sa

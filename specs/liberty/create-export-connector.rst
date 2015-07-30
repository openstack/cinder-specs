..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add connector object to create_export call
==========================================

https://blueprints.launchpad.net/cinder/+spec/add-create-export-connector

There are a good number of volume drivers in Cinder today, that are using
initialize_connection to do the work of exporting a volume as a target
to an initiator (nova compute host).  One of the reasons that those drivers
are doing that, is because the create_export method API doesn't include
the connector object being passed in.  This spec outlines adding the
connector object to the standard volume driver API for create_export.

Problem description
===================

There are many drivers in Cinder today that are doing volume target exports
inside of initialize_connection, instead of the create_export method.
One of the reasons for this is a documented understanding of what the
differences between these 2 methods is supposed to be.  The other problem
that prevents drivers from doing the work inside of create_export, is that
create_export is missing the connector object coming from the initiator.
Some backends require the connector object, which contains the initiator
information, in order to export a target from their array.  So, drivers
end up moving all of that exporting code into initialize_connection, because
that's the only driver call that has the connector object.

The other issue with Cinder drivers doing the exporting of volume targets
in initialize_connection, is that Nova calls initialize_connection repeatedly
during other processes not involved with attaching a volume, such as live
migration.  Nova will call a driver's initialize_connection simply to fetch
the connection_information about an exported volume, not because it wants
to attach a volume.

Use Cases
=========

The main use case is attaching a volume.  When someone calls Cinder to attach
a volume, they eventually call the volume manager's initialize_connection
method.  The volume manager's initialize_connection method first calls
create_export, with the intention of asking the driver to create a target
that is exported from the array to an initiator.  Then the volume manager
calls the driver's initialize_connection to fetch the target information to
pass back to the caller, to discover the volume showing up.


Proposed change
===============

The proposed change is simple.  Add the already existing connector object
in the call to create_export.  This makes the connector object universally
available to all drivers during create_export time.  Then each driver can
eventually be changed by each maintainer to do the target exporting, instead
of during initialize_connection time.


Alternatives
------------

We can do nothing, and ask that each driver maintainer be more careful about
doing any creating of exports inside of initialize_connection if they already
exist.  This basically makes create_export useless and a noop for all of those
drivers.  I'd rather see every driver use each method as they were intended, so
to make reviewing drivers more consistent accross all of Cinder.


Data model impact
-----------------

None

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

If driver developers move the creating of the exports into create_export,
then any calls into initialize_connection should be faster, depending on the
driver of course.

Other deployer impact
---------------------

None

Developer impact
----------------

Once the connector object is available inside of create_export, then it's
possible for drivers to migrate their existing export creation code from
initialize_connection into create_export.  Reviewers should also be looking
for this in existing and future reviews, and not approving drivers that ignore
create_export.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  walter-boring

Work Items
----------

First is to update the base driver.py's create_export method.
Then each other driver that defines create_export will have to accept the
new connector object.

Then the volume manager call to create_export will need to pass the connector.


Dependencies
============

None


Testing
=======

Any and all Unit tests will need to be updated to note the new connector
object parameter.  Driver's unit tests will need to be updated as well.


Documentation Impact
====================

None


References
==========

A lot of research has gone into this, due to the efforts of trying to
make Nova live migration with cinder volumes work.  There is an existing
etherpad that talks about the known issues of Nova, Cinder interaction.
That etherpad also lists out the Cinder drivers that have potential problems
with live migration due to creating exports inside of initialize_connection.

* https://etherpad.openstack.org/p/CinderNovaAPI

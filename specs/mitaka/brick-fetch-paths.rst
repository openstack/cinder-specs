..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Brick fetch volume paths API
==========================================

https://blueprints.launchpad.net/cinder/+spec/brick-fetch-paths

The Brick (os-brick) library currently has the knowledge of how
to discover and remove volumes from the scsi subsystem on a host.
Every connector object has the ability to return a list of existing
paths on the system for a given volume, as well as the location
where those volume paths live.   This spec's purpose is to detail
a formal API for fetching the existing paths for a volume, as well
as the search path, and the list of all existing volumes for a
connector.   The purpose of this is to allow the verification of
volume paths after attachment, as well as verification of a volume
being removed after detachment.   Once we have this API in place,
then it's possible to automate the verification of attaches and
detaches for complex operations.


Problem description
===================

Historically in OpenStack, we haven't had a way of automatically
verifying that the detach process doesn't leave behind orphaned
volumes.  I define orphaned volumes as volume paths on a system
that were associated with a volume, that has been detached.

This can cause problems later on when volumes are attached to that
host.  Some SAN arrays reuse iqn's and or LUN ids for volumes.
This can get a host in the state where a volume on the host conflicts
with an orphaned volume path on the host, leading to corruption with
i/o.


Use Cases
=========

The primary use case is for fetching the list of existing paths for a volume
on a system after attach and detach.

Proposed change
===============

Add new APIs to each os-brick connector to
1) fetch the existing paths on the system for a given volume.
2) fetch all of the volume paths for a connector.

Flask based API script.
Add a new flask script that exposes these API calls that can be run
in a test environment.  This flask script is needed in order to
automatically verify the success of detach during nova live migration
which happens on multiple compute hosts.   This flask script is only
intended on being used during jenkins tests and gate checking.

Alternatives
------------

Someone can write individual verification scripts, outside of os-brick,
for each type of volume (connector), but won't benefit from future fixes
in connectors.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

os-brick already returns a single path entry after connect_volume calls.
What os-brick doesn't do today is return a list of all volume paths
associated with a connector.

It's noted here that the flask script as described in the proposed change,
should not be run/used on a production/live system.  This is intended for
check/gate testing only.


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

Connector developers will be required to implement the new APIs for their
connector, even if they are stubs.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  walter-boring

Other contributors:
  anthony-mic-lee

Work Items
----------

I have a Work In Progress patch already up in gerrit for the API work.
We need to add a follow up patch that creates the flask script for remotely
calling the methods for the connector.

Dependencies
============

None

Testing
=======

Standard unit tests.
I have also done manual testing with it.  Eventually, we would like to add
verification in check/gate testing for volume attaches as well.
For each connector object we should have 3rd party CI to test the new APIs
as well.


Documentation Impact
====================

The code is documented.


References
==========

The existing patch in gerrit
* https://review.openstack.org/#/c/199764/

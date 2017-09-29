..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add shared_targets column to volumes
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/add-shared-targets-flag-to-volume

One problematic thing about dealing with device attachments is that some
devices share a single target for multiple volumes while others have a
single unique target for every volume.

For example, a device utilizing shared targets, may have a single iSCSI
connection shared for multiple volumes attached to a Nova compute node.

This can be troublesome because extra care needs to be taken by consumers to
make sure that the target it's not being used by another volumes on a system when
a delete/detach call is made.

Problem description
===================

Currently when os-brick receives a detach operation it attempts to inspect the
system and determine if there are multiple volumes attached to the host sharing
the same target.  Most of the time this works, however it tends to be a bit
prone to races, additionally attempting bus scans is not only inefficient but
unreliable.

The other problem is that currently there is no indication for os-brick to know
if it's dealing with a device that utilizes shared targets or not, so we go
through this scanning routine even if we don't necessarily need to.

The process of removing targets would be much more robust if we actually knew
ahead of time if the volume was hosted on a device that utilizes shared
targets, in which case we could do things more efficiently including using
locks on a compute node around attach/detach operations for a specific backend.

Use Cases
=========

This improves the existing use case of detaching volumes correctly and
efficiently.  The basic condition is that you have a device that utilizes
shared-targets, and you have a single volume (V-A) from that device attached to
a compute node.  A user issues a detach call and at the same time another user
issues an attach of a different volume (V-B) from the same backend that will land on
the same compute node.

It's possible that the connection response for V-B is completed under the
assumption that the target exists (due to V-B) but then the V-A detach process
performs it's checks prior to the attachment of V-B being finalized, and as
a result it removes the target and the connection of V-B will appear to have
been successful but the volume will not be accessible.

Proposed change
===============

Add a column to the volume reference that indicates if the backend associated
with the volume utilizes shared-targets or not.  In addition add
a backend-name member to the volume detail view.  When shared-targets is True,
the consumer will know that they're dealing with a device that uses
shared-targets and that it needs to take some extra precautions.

In addition we need to provide a unique identifier for the backend, while at
the same time not leaking details of the abstraction to end users.  The
service UUID should work nicely for this.

For the case of folks that use a single device to back multiple c-vol
services we'll provide the ability to over-ride/set this identifier in the
config file.  That way an admin can choose to set multiple c-vol configurations
to use the same backend_id if needed.

Note that by default we will set the shared_targets flag to True.  We use this
as the default because there's no harm or functional issues with treating
unique targets as shared, it's just going to introduce some unnecessary locking
but will not impact functionality.  Conversely if a device that actually uses
shared targets is False and not locked we have a race condition between attach
and detach calls that causes problems.

Alternatives
------------

The scan method we use today works most of the time and is reasonable.  We
could certainly just continue to use that by itself.

Data model impact
-----------------

This change adds a new boolean column to the Volumes table (shared_targets).

* What new data objects and/or database schema changes is this going to
  require?

  This introduces a new column to the Volumes Table

* What database migrations will accompany this change.

  We'll perform a DB migration that adds the column and sets its value to
  True.  Setting to True is safe for non-shared targets so while it's not
  necessarily the most efficient way to do things, it's safe and reliable.

* How will the initial set of new data objects be generated, for example if you
  need to take into account existing volumes, or modify other existing data
  describe how that will work.

  During init of the volume service, we'll query the backend capabilities for
  the setting "shared_targets", if the value from the stats structure is False
  we'll check any volumes that are set as True and update them accordingly.
  This way we migrated everything and set it to true, then on service init we
  verify that the setting is actually correct.

  NOTE that we're already iterating over the volumes of a backend on init, so
  we're not introducing any new overhead or lag here.

REST API impact
---------------

This change does not impact the arguments or options for any of the existing
API calls, however it does add additional fields to the Volume Get response.

When the appropriate micro-version is selected, we'll add two additional fields
to the detailed volume response view:
    shared_targets
    backend_name

Security impact
---------------


Notifications impact
--------------------


Other end user impact
---------------------

This change mostly just provides the user with additional info.  It's up to the
user/consumer to determine if they want to do anything with this extra info or
not, but it does not change their current use-cases or workflow.

Performance Impact
------------------

Other deployer impact
---------------------

Developer impact
----------------

Be default we'll set the shared_targets flag for volumes to True, but driver
maintainers should add the appropriate stats field to their driver to change
this if they don't use shared_targets.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    john-griffith

Work Items
----------

Add the changes to Cinder and bump the max support MV in cinderclient.

Dependencies
============

Testing
=======

Add a functional test for the specific microversion and ensure the appropriate
response.

Documentation Impact
====================

Documenting the new fields in the detailed volume response, and also recommend
how it can be used.  Or just reference this spec.

References
==========


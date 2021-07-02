..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Reset State Robustification
===========================

https://blueprints.launchpad.net/cinder/+spec/reset-state-robustification

Problem description
===================

I have seen a number of users get volumes into "invalid" states by
not understanding how resetting the state of a volume interacts
with resetting the state of attachments.

For example, on a volume with an attachment, run
"cinder reset-state --state available <vol>"

What now?

Use Cases
=========

Help prevent users who aren't Cinder experts from shooting themselves
in the foot with reset-state.

Proposed change
===============

If a user requests to reset the state of a volume to something that
Cinder knows is not a valid state of the universe, reject the request
with a reason.

If volume1 has an attachment active.
$ cinder reset-state --state available volume1
ERROR: Cannot reset-state to available because attachments still exist.

$ cinder reset-state --state available volume1 --attach-status detached
(command succeeds)

Cases to block:
   1. volume reset to available w/ attachments
   2. snapshot reset to in-use with volume in available


Sometimes a knowledgeable operator may need to reset the state anyway
and then manually make the current state valid. cinder-manage is the
place where forcing a change will be allowed instead of '--force'
flag in the API.

Alternatives
------------

- Hope users don't do these things.
- Handle the "reset to available while attached case" by forcefully
  detaching the volumes instead of rejecting the request.

Data model impact
-----------------

None

REST API impact
---------------

os-reset-status actions on volumes, snaps, backups, groups,
will now return a 400 in some cases where they would previously
succeeded.  This does not require a microversion bump.

Security impact
---------------

None

Active/Active HA impact
-----------------------

These checks in reset-state could concievably race against updates in
a cluster.  Will determine what that means when we get further into
implementation.


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

Improves safety for deployers trying to clean up from issues in
their cloud.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  TusharTgite

Other contributors:
  eharney

Work Items
----------

* Implement a check for the
  reset-state to available while attached case
* Add code to cinder-manage to handle the case
  where an operator needs to override the API
  and reset the state anyway.
* Research other sensible cases we could prevent for
  volumes, snaps, groups, etc.
* Tempest test for a couple of the big cases


Dependencies
============

None

Testing
=======

* New tempest tests


Documentation Impact
====================

* Should document common cases where this fails.

References
==========

* Original proposal: https://review.opendev.org/c/openstack/cinder-specs/+/682456

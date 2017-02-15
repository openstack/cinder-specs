..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Client reset-state improvements
===============================

https://blueprints.launchpad.net/cinder/+spec/client-reset-state-improvements

Improve "reset-state" in the cinder client shell.


Problem description
===================

The reset-state implementation in the client shell has a few issues
that merit cleaning up:

We have and are adding an <x>-reset-state operation for many objects
in Cinder.  This results in numerous commands when we could consolidate
these into one command.  Consolidating things into a single command
makes more sense because this is intended to be a rarely-used admin
fix-up tool, and not a prominent feature of our client.

The current reset-state model also defaults to unsafe behavior.  It
will reset an object to "available" if a user runs it on an object
without other parameters.  This is not a safe path to use as a default
because we have no way to know if the object is actually in an
"available" state and resetting it to "available" will result in odd
problems later.


Use Cases
=========

This functionality addresses admins who need to repair broken
things in Cinder to workaround bugs or failed operations.

Proposed change
===============

* Add a new argument to "cinder reset-state" so that it can handle
  all of the types of objects we would like to reset the state of.

.. code-block: bash

    $ cinder reset-state --type volume --state available volume-abcd

    $ cinder reset-state --type snapshot --state error abcd-1234

    $ cinder reset-state --type backup --state error esdf-5678

- See https://review.openstack.org/#/c/413156/


* Require state to be specified rather than defaulting to "available".

This is trickier to keep from breaking compatibility, but worthwhile.
The defaulting is done in the client.  We could handle this by adding
a stricter check in the client using the microversion checks, so that
using microversion 3.30+ requires the user to specify the desired
state, but when using older API versions, the previous behavior
is kept.  This does not correspond with an actual API change
on the server, but should work as a way to handle compat here.


* Decide that we will no longer add <x>-reset-state operations to
  the client shell.


* (Optional:) We should consider looking at attach-status and
   migration-status resets as well.  These are required to be specified
   manually but there may be cases where it would be more consistent
   to handle this for the caller.

   i.e. does it ever make sense to reset a volume to "available" and not
   clear its attach status?


Alternatives
------------

* Leave things as they are and keep adding new reset operations to the client
  which default to hazardous behavior.

* Start fixing things in Cinder that require use of reset state in the first
  place.  (We should do this, but it's a long, hard, project, and out of scope
  here.)


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

The cinderclient CLI changes.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

* We should design Cinder to not require use of reset-state as a
  "regular" thing.

  Currently there's a model, for some operations, of:
    1. Request operation
    2. Operation fails
    3. User is notified that the operation failed by the object
       now being in "error" state.
    4. For some operations, the admin is now expected to reset
       the state back to "available" to keep things functioning.

  This could be done as:
    1. Request operation
    2. Operation fails
    3. Object is put back into previous state instead of "error"
    4. Client polls for operation status via our async messaging
    5. User now knows what happened and the admin doesn't have to
       perform a reset.  The object is left in its "true" state.


* Should consider what the intersection of reset-state and force-detach
  looks like.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  eharney

Other contributors:
  tommylikehu


Work Items
----------

* Continue on https://review.openstack.org/#/c/413156/

Dependencies
============

None

Testing
=======

Covered by unit tests in the cinder client.


Documentation Impact
====================

Cinder shell client documentation will need updates.


References
==========

* cinderclient change: https://review.openstack.org/#/c/413156/

* reset-state in openstackclient: https://review.openstack.org/#/c/268907/

* Obsoletes some of
  pike/support-reset-generic-group-and-group-snapshot-status.rst

* IRC meeting discussion: ``http://eavesdrop.openstack.org/meetings/cinder/2017/cinder.2017-01-04-16.02.log.html#l-153``

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Valid States API
================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/valid-states-api

Provide an API to obtain the set of valid states that are permissible to be
used in the function to reset the state of a volume and snapshot.

Problem description
===================

The purpose of this feature is to facilite exposing the reset-state API in
horizon in a meaningful way by restricting the set of permissible states that
the administrator can specify for a volume.  There is no API for this, and it
is undesirable to hardcode this information into horizon.

Use Cases
=========

Proposed change
===============

A new API function and corresponding cinder command will be added to determine
the set of valid states for volumes or snapshots.

The initial proposal is to create a single function, get_valid_states, to
obtain the valid states for any type of resource (volume, snapshot).

Alternatives
------------

For consistency with the rest of cinder, get_valid_states may be renamed and/or
split into multiple functions, one per resource type; this decision will be
left as an implementation detail and will be finalized as part of the normal
code review process.

Data model impact
-----------------
None

REST API impact
---------------

Add a new REST API to retrieve valid states:
  * GET /v2/{tenant_id}/states

JSON response schema definition::

    'valid_states': {
        'type': 'array',
        'items' : {
            'type': 'string'
        }
    }

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------

A new command, get-valid-states, will be added to python-cinderclient.  This
command mirrors the underlying API function.

Obtaining the list of valid states for a volume or snapshot can be performed
by:
$ cinder get-valid-states


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

Primary assignee:
  thingee

Work Items
----------

* Implement REST API
* Implement cinder client functions
* Implement cinder command

Dependencies
============

Horizon blueprints that will depend on this one:

* https://blueprints.launchpad.net/horizon/+spec/cinder-reset-volume-state

* https://blueprints.launchpad.net/horizon/+spec/cinder-reset-snapshot-state

Testing
=======
None


Documentation Impact
====================

The cinder client documentation will need to be updated to reflect the new
command.

The cinder API documentation will need to be updated to reflect the REST API
changes.


References
==========

None

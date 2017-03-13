..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Backup capability
==========================================

https://blueprints.launchpad.net/cinder/+spec/discovering-system-capabilities

The spec [1] made a decision that a new API will be created for users or other
projects to discover system capabilities of Cinder. This spec is a follow-up,
and defines the new interface.

Problem description
===================
Currently Horizon knows whether backup services are enabled through a config
setting “enable_backup" which needs to set manually. This is not only
cumbersome but also prone to operator error.

Use Cases
=========

Horizon can do some queries to know whether backup services are enabled without
manually setting config file.

Proposed change
===============

A new API will be added to query system capabilities. At first, the result only
contains whether backup services are enabled.

Alternatives
------------

Please see [1].

Data model impact
-----------------

None.

REST API impact
---------------

Add a new API to query system capabilities. At first only backup is included.

.. code:: javascript

    GET /capabilities

    {
        "service": {
            "volume-backups": {
                "available": true
            }
        }
    }

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

User can use the interface to query backup capability.

Performance Impact
------------------

New API and should not impact existed APIs.

Other deployer impact
---------------------

Operator no longer has to manually set “enable_backup” in Horizon config
settings file. Back-compat story for this Horizon config change is out of
scope for this spec.


Developer impact
----------------

Other capabilities can be added later if needed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lixiaoy1

Work Items
----------

* Add API to query system capabilities, currently only backup.
* Add test cases.
* Update API docs.

Dependencies
============

None.

Testing
=======

Unit, functional and tempest test cases will be added to validate this new
API.

Documentation Impact
====================

* New API and client call in Cinder needs to be documented.

References
==========

..[1] https://specs.openstack.org/openstack/cinder-specs/specs/newton/discovering-system-capabilities.html

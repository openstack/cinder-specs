..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Promote Secondary Backend After Failover (Cheesecake)
=====================================================

https://blueprints.launchpad.net/cinder/+spec/replication-backend-promotion

Problem description
===================

After failing backend A over to backend B, there is not a mechanism in
Cinder to promote backend B to the master backend in order to then replicate
to a backend C.

Current Workflow
----------------
1. Setup Replication
2. Failure Occurs
3. Fail over
4. Promote Secondary Backend
  a. Freeze backend to prevent manage operations
  b. Stop cinder volume service
  c. Update cinder.conf to have backend A replaced with B and B with C
  d. *Hack db to set backend to no longer be in 'failed-over' state*
    * This is the step this spec is concerned with
    * Example:
      ::

        update services set disabled=0,
                            disabled_reason=NULL,
                            replication_status='enabled',
                            active_backend_id=NULL
          where id=3;
  e. Start cinder volume service
  f. Unfreeze backend

Use Cases
=========
There was a fire in my data center and my primary backend (A) was destroyed.
Luckily, I was replicating that backend to backend (B). After failing over
to backend B and repairing the data center, we installed backend C to be a
new replication target for B.

Proposed change
===============

Add the following commands to reset the active backend for a host.
::

    cinder reset-active-backend <backend-name>
    PUT /os-services/reset_active_backend {"host": <backend-name>}

Equivalent to:
::

    update services set disabled=0,
                        disabled_reason=NULL,
                        replication_status='disabled',
                        active_backend_id=NULL
       where host='<backend-name>;

The following to re-enable replication from backend B -> C.
::

    cinder replication-enable <backend-name>
    PUT /os-services/replication_enable {"host": <backend-name>}

Equivalent to:
::

    update services set replication_status='enabled'
       where host='<backend-name>;

Alternatives
------------

Instead of 'admin' APIs, we add cinder-manage commands.
::

    cinder-manage reset-active-backend <backend-host>
    cinder-manage replication-enable <backend-name>

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
  ?

Work Items
----------

* Implement cinder reset-active-backend API
* Implement cinder replication-enable API
* Document post-fail-over recovery in Admin guide

Dependencies
============
None

Testing
=======

None

Documentation Impact
====================

Documentation in the Admin guide for how to perform a backend promotion.

References
==========

None

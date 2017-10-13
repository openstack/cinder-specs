..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Migrate ConfKeyManager encryption key ID into Barbican
======================================================

https://blueprints.launchpad.net/cinder/+spec/migrate-fixed-key-to-barbican

Support the transition to Barbican by migrating existing keys associated with
the ConfKeyManager into Barbican. The ConfKeyManager key uses a key ID of all
zeros that's associated with the fixed_key configuration parameter in the
key_manager section of cinder.conf.


Problem description
===================

The legacy ConfKeyManager is insecure, and the prefered encryption key manager
is Barbican. Using Barbican in new deployments is easy, but switching
deployments from the ConfKeyManager to Barbican is difficult when there are
existing volumes encrypted using the ConfKeyManager.


Use Cases
=========

A user has a deployment based on the ConfKeyManager. They wish to to switch to
Barbican, but there are existing volumes encrypted using the ConfKeyManager.
These existing volumes must continue to function when Barbican is the key
manager. The user needs a migration path to Barbican that doesn't break
existing encrypted volumes.


Proposed change
===============

This spec proposes that all instances of the ConfKeyManager's key (identifed
by an all-zeros key ID) be replaced with an identical key stored in Barbican.
This is possible because Barbican provides a mechanism for storing existing
encryption keys.

A cinder-volume thread will be used to search for encryption key ID instances
in the database that match the ConfKeyManager's all-zeros ID. This thread will
be refered to as the key migration thread.

* The key migration thread will search the volume table for encryption_key_id
  entries matching the all-zeros ConfKeyManager key ID. For each matching
  entry in the volume table, a new Barbican key ID will be created with the
  exact same secret derived from the fixed_key config value used by the
  ConfKeyManager. The volume table is updated with the new Baribican key ID.

* The thread will only migrate keys associated with volumes belonging to that
  host. This will avoid contention when there are multiple cinder-volume hosts.

* The key migration thread will also look for ConfKeyManager keys stored in
  metadata tables. Where appropriate, these entries will be updated with the
  Barbican key ID stored in the corresponding volume table. Otherwise, a new
  Barbican key ID will be created.

* The key migration thread will be spawned on start-up whenever both the
  following conditions are true:

  1. The ConfKeyManager's "fixed_key" is configured
  2. The key_manager's "backend" (was "api_class") is *not* the ConfKeyManager

* The key migration thread will generate logs that indicate a key ID has
  been migrated to Barbican. It will also generate a log that indicates the
  thread has run to completion without finding any key IDs associated with
  the ConfKeyManager.

The user is responsible for deciding when to remove the fixed_key from the
deployment. It is recommended the user use the logs messages described above
to determine when the key migration process has completed. Once the user
removes the fixed_key from the configuration, the key migration thread will
not be spawned on startup.

The key migration thread will work in conjunction with an enhancement to
Castellan and/or Barbican. During the migration process, cinder services may
still encounter the ConfKeyManager's key ID (all zeros) in the database. This
is not a valid Barbican key ID, and Barbican will throw an error if asked to
provide the key associated with that ID. The enhancement will essentially
intercept requests for the all-zeros key ID, and return the ConfKeyManager's
response. All of this would be totally transparent to the rest of the Cinder
code. In short, the all-zeros key ID will continue to work even with Barbican
as the designated key manager.

Alternatives
------------

Instead of using a separate migration thread, each piece of code that fetches
and uses the encryption key ID could be responsible for migrating the key into
Barbican. However, there are multiple places where the code needs to handle
the key ID, and each instance would need to be updated to perform the
migration. This approach would involve significantly more code change, and
would require more long-term maintenance.

We could also not migrate keys to Barbican at all, and simply allow the
Castellan-Barbican enhancement to handle requests for the all-zeros key ID.
However, this would require the user retain the fixed_key value in the
configuration, which essentially misses the point of migrating to Barbican.


Data model impact
-----------------

None. Existing database entries that contain the ConfKeyManager's encryption
key ID will be updated with references to a Barbican key ID, but the database
structure will not change.

REST API impact
---------------

None.

Security impact
---------------

This change should be considered a security enhancement in that it facilitates
transitioning existing ConfKeyManager deployments to Barbican.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

Minimal. The key migration thread will not be CPU intensive, and other cinder
operations will not need to block until the migration thread completes.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Alan Bishop<abishop@redhat.com>

Work Items
----------

* Develop code framework for scanning tables for encryption_key_id that needs
  to be migrated to Barbican
* Develop utility code for generating new Barbican key ID with ConfKeyManager's
  secret
* Add/update unit tests
* Update documentation


Dependencies
============

Castellan and/or Barbican, will be enhanced to aid the migration process. The
enhancement will allow volumes encrypted with the ConfKeyManager's key ID to
function properly during the migration process.


Testing
=======

- Add and update the unit tests.


Documentation Impact
====================

The documentation will be updated to state that volumes encrypted using the
ConfKeyManager will continue to function correctly after switching to
Barbican. The user needs to feel comfortable knowing the migration to Barbican
is seamless and automatic.

The changes will explain the user is responsible for removing the
ConfKeyManager's fixed_key from the deployment, with the recommendation that
the decision to do so be guided by the log messages generated by the key
migration thread. Those message will indicate when keys are still being
migrated, and when the migration process has finished.


References
==========

None.

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Support deferred volume deletion in the RBD driver
==================================================

https://blueprints.launchpad.net/cinder/+spec/rbd-deferred-volume-deletion

This spec proposes configurable support for deferred volume deletion in the
RBD driver, using the trash_* API calls, as introduced with the Luminous
version of Ceph.

Problem description
===================

Volume deletion in Cinder is synchronous: the c-vol worker thread deleting
a volume will return only when the backend confirmed the removal of the
volume. This approach ensures that the space used on the backend will at
most be equal to the quota of the corresponding project, and hence allows
for proper resource management. The current implementation of the RBD
driver follows this approach by using RBD's 'remove()' API call.

For deployments with thousands of volumes/users and large volumes (10-100TB)
this can create two potential issues:

1) a c-vol worker thread will be blocked during deletion and depending
   on the size of the volume, the speed of the backend, the number of
   concurrent requests, and the total number of worker threads, this may
   block other operations, makes the service seem unresponsive, and can
   lead to timeouts.

   [1] was proposed to mitigate this problem by making the number of
   native threads configurable and profit more efficiently from the
   backend's potential capabilities.

2) users have to wait until the deletion is completed on the backend
   before they can re-use the freed up quota/space -- despite having
   signalled to Cinder that the volume should be removed and the quota
   be freed. For large volumes, this waiting time can be significant
   (hours for large volumes) and has hence a corresponding negative
   impact on the user experience.

To address these issues, this spec proposes to optionally support
asynchronous volume deletion in the RBD driver.

Use Cases
=========

Administrators can optionally enable a deferred deletion: the Ceph backend
will acknowledge the deletion immediately adressing the two issues above.

Proposed change
===============

The proposal is to

- amend the RBD driver to support deferred deletion by optionally calling
  RBD's 'trash_move()' rather than RBD's 'remove()' in the driver's
  implementation of 'delete_volume()';

- add a periodic timer in the RBD driver to trigger a regular purge.

For this, the following additional config options would be needed:

- 'enable_deferred_deletion' (default: 'False')
   If enabled, RBD's 'thrash_move' will be called rather than 'remove'.

- 'deferred_deletion_delay' (default: 0)
   Number of seconds before an image moved to trash is regarded as expired
   and can be purged.

- 'deferred_deletion_purge_interval' (default: 3600)
   Number of seconds between runs of the periodic timer to purge the
   backend's trash.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None.

Cinder-client impact
--------------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

For the end users, deletions will appear to be instantaneous. The freed
up quota will become available immediately.

Performance Impact
------------------

Volume deletions will become independent from the backend's speed or the
size of the volumes.

Other deployer impact
---------------------

As the deletion is deferred, the space occupied will not be freed immediately.
Deployers need to be aware that the maximum space needed can exceed the sum
of the allocated quota of all projects.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Arne Wiebalck (arne.wiebalck@cern.ch)

Work Items
----------

* Update RBD driver to respect options for deferred deletions and
  call 'trash_move' if enabled.
* Add periodic task to call 'trash_purge' for backend which have
  deferred deletion enabled.
* Add related unit testcases.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

* Add administrator documentation to advertise the option of deferred
  deletions for RBD and explain why&when&how this should be used,
  emphasizing in particular that by enabling this option the space of
  deleted volumes will not be freed on the RBD backend until the trash
  is purged.

References
==========

_`[1]`: https://review.openstack.org/#/c/550858/

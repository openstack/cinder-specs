..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Support create volume from backup
=================================

https://blueprints.launchpad.net/cinder/+spec/support-create-volume-from-backup


This blueprint proposes support for creating a volume from backup.

Problem description
===================

Cinder has supported create volume from volume, snapshot and image.
But when considering create volume from backup, users have to create a
new volume first and then restore the volume with backup. Although
we can restore backup without volume id, in that case Cinder will
create a new volume in the background, but we still have some issues
because we don't have any access to control the volume's az, volume
type, size and many other attributes.

Use Cases
=========

User can directly create volume from backup rather than create a new volume
and then restore it.

Proposed change
===============

This spec proposes to support create volume from backup, after this change
users can directly create a new volume with backup id as below:

.. code-block:: bash

    $ cinder create [size] --backup-id <id> [other_options]

Volume's size is equal to the backup's original volume size if size
option is omitted.

Since we already introduced the generic backup implementation `[1]`_ for our
existing volume drivers, it makes sense to add this ability to our volume
drivers, so when create volume from backup, the request will be scheduled
to the volume backend as normal, and first we would try the vendor-specific
``create_volume_from_backup`` method as below:

.. code-block:: python

    def create_volume_from_backup(self, volume, backup):
        """Creates a volume from a backup."""

        raise NotImplementedError()

If the driver reports ``NotImplemented`` or ``NotSupported``, Cinder will
directly create the raw volume at the backend and then schedule the request
to the backup service to restore the volume with backup.

One of the differences between creating volume from snapshot and
creating volume from backup is the latter could be time consuming, so we need
to update the backup's status to prevent possible usages by other requests.
For instance, when creating volume from backup the possible status for volume
and backup are::

    1. Volume: creating, available, error
    2. Backup: restoring, available, error

**Note**: We will use 'creating' instead of 'restoring-backup' for the
volume's transition status here in order not to expose the creating detail.

Alternatives
------------

Keep using current restore API or create and restore mechanism to cover
this use case.

Data model impact
-----------------

None

REST API impact
---------------

Microversion bump is required for this change.

Cinder-client impact
--------------------

Cinder-client will be updated to support create volume from backup.

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

When creating a volume from backup and the vendor-specific method does
not exist, the creation process could take a little longer than
a typical creation due to the restore action.

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

  1. luqitao (qtlu@fiberhome.com)
  2. tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Support create volume from backup in Cinder.
* Add related unit testcases.
* Add related tempest testcase.
* Update cinder-client and OSC.

Dependencies
============

Depends on generic backup implementation `[1]`_

Testing
=======

* Add unit tests to cover creating volume from backup.

Documentation Impact
====================

Both API documentation and CLI documentation should be updated.

References
==========

* _`[1]`: https://github.com/openstack/cinder-specs/blob/master/specs/queens/generic-backup-implementation.rst

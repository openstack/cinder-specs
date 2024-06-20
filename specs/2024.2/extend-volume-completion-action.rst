..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Add extend volume completion action
===================================

https://blueprints.launchpad.net/cinder/+spec/extend-volume-completion-action

This blueprint proposes a new volume action that can be used by Nova to notify
Cinder on success or failure when handling ``volume-extended`` external server
events.
The new volume action is used to add support for extending attached volumes to
the NFS, NetApp NFS, Powerstore NFS, and Quobyte volume drivers.

Problem description
===================

Many remotefs-based volume drivers in Cinder use the ``qemu-img resize``
command to extend volume files.
However, when the volume is attached to a guest, QEMU will lock the file and
``qemu-img`` will be unable to resize it.

In this case, only the QEMU process holding the lock can resize the volume,
which can be triggered through the QEMU monitor command ``block-resize``.

There is currently no adequate way for Cinder to use this feature, so the NFS,
NetApp NFS, Powerstore NFS, and Quobyte volume drivers all disable extending
attached volumes.

Use Cases
=========

As a user, I want to extend a NFS/NetApp NFS/Powerstore NFS/Quobyte volume
while it is attached to an instance and I want the volume size and status to
reflect the success or failure of the operation.

Proposed change
===============

Nova's libvirt driver uses the ``block-resize`` command when handling the
``volume-extended`` external server event, to inform QEMU that the size of an
attached volume has changed.
It is in principle also capable of extending a volume file, but is currently
unable to provide feedback to Cinder on the success of the operation.

Currently, Cinder will send the ``volume-extended`` external server event to
Nova only after it has finalized the extend operation and reset the volume
status from ``extending`` back to ``in-use``.

This spec proposes to give volume drivers a mechanism to hold off finalizing
the extend operation until after the ``volume-extended`` event has been sent
and Cinder has received feedback from Nova that it was handled successfully.

This spec also proposes a new volume action that Nova will use to provide this
feedback to Cinder.

API
---

A new API microversion is introduced, adding the new
``os-extend_volume_completion`` volume action.

The volume action takes a boolean ``error`` argument, indicating success or
failure to extend the attached volume.
It is intended to be used exclusively by Nova to notify Cinder, and an
appropriate policy will be added to enforce this.

API.extend_volume_completion
----------------------------

The new volume action will be handled by a new method in the volume API:

.. code-block:: python

    def extend_volume_completion(self,
                                 context: context.RequestContext,
                                 volume: objects.Volume,
                                 error: bool) -> None:

The new method expects the volume to have status ``extending``, and to have the
keys ``extend_reservations`` and ``extend_new_size`` in its admin metadata.
The first should hold a list of quota reservations, and the second should
contain an integer larger than the volume's current size, representing the new
size after extending.

If these conditions are not met, then an ``InvalidVolume`` exception will be
raised, resulting in an HTTP response of ``400 Bad Request``.

If the conditions are met, it will remove size and reservations from the admin
metadata and call ``VolumeManager.extend_volume_completion()`` via RPC,
passing both as arguments.

VolumeManager.extend_volume_completion
--------------------------------------

.. code-block:: python

    def extend_volume_completion(self,
                                 context: context.RequestContext,
                                 volume: objects.Volume,
                                 new_size: int,
                                 reservations: list[str],
                                 error: bool) -> None:

The behavior of this method depends heavily on the ``error`` argument:

* If ``error`` is ``True``, the method will roll back the quota reservations,
  set the volume status to ``error_extending``, and log the error.

* If ``error`` is ``False``, it will finalize the quota reservation, update
  the size field of the volume to the new size, and reset the volume status to
  ``available`` or ``in-use``, depending on the presence of attachments.
  It will also update the pool stats and send a ``resize.end`` notification
  with the new volume size.

This is identical to how ``VolumeManager.extend_volume()`` currently handles
success and failure of the volume driver's ``extend_volume()`` method, except
that this method will not notify Nova with the ``volume-extended`` external
server event.

VolumeDriver.extend_volume
--------------------------

A mechanism will be introduced by which the driver's ``extend_volume()``
method can signal to the volume manager that it has to wait for a response
from Nova before finishing the extend operation.
This could take the form of a return value or a new exception that the volume
manager will have to catch.

The NFS, NetApp NFS, Powerstore NFS, and Quobyte volume drivers currently
have checks in their respective ``extend_volume`` methods, that will raise an
exception if the volume to be resized is attached, causing the operation to
fail.
Those checks will be removed.

Instead, the drivers will catch any exceptions resulting from the volume files
being locked (see the proposed change to ``nfs.py`` in [1]_ for an example on
how to do that), and notify the volume manager that feedback from Nova is
required.

VolumeManager.extend_volume
---------------------------

The call to the volume driver's ``extend_volume()`` method will be handled as
follows:

* If the call fails, ``extend_volume_completion`` will be called with
  ``error=True``.

* If the call succeeds, but the volume is not attached,
  ``extend_volume_completion`` will be called with ``error=False``.

* If the call succeeds, and the volume is attached,
  ``extend_volume_completion`` will be called with ``error=False`` and Nova
  will be notified with the external server event.

This matches the current inline behavior of the method, and covers offline
extend for all drivers, as well as online extend for the drivers that
previously supported it.

To support remotefs-based drivers that have to rely on Nova for online extend,
two aditional cases will be handled:

* If the driver notifies the volume manager that a response from Nova is
  required, but the volume is not attached, or the volume is attached to more
  than one instance, it will be handled as failure and
  ``extend_volume_completion`` will be called with ``error=True``.

  QEMU can not resize shared volume files, because they are locked read-only,
  so adding multi-attach support for this feature is currently not worthwhile.
  However, support may be added later if other drivers require it, e.g. by
  enabling Cinder to handle multiple completion actions for the same volume.

* If the driver notifies the volume manager that a response from Nova is
  required, and the volume is attached to exactly one instance, then Cinder
  will store the quota reservations and the target size in the in the admin
  metadata with the keys ``extend_reservations`` and ``extend_new_size``.

  It will then attempt to send the ``volume-extended`` external server event
  with the new Nova API microversion proposed in [4]_, making sure that Nova
  supports using the ``os-extend_volume_completion`` action.

  * If the ``volume-extended`` event has been submitted to Nova successfully,
    this method will just return normally.
    The volume will now be left in status ``extending``, which will signal to
    Nova that it should respond with the ``os-extend_volume_completion``
    action, as described in the `Nova`_ subsection.

  * If the ``volume-extended`` event could not be submitted, the operation
    will be rolled back by calling ``extend_volume_completion`` with
    ``error=True``.

    This can happen if Nova doesn't support the required microversion yet, or
    if the external event API responded with an error code such as ``403`` or
    ``404``.

Visible Admin Metadata
----------------------

``extend_new_size`` has to be stored in the admin metadata, because the
regular volume metadata is editable by users.
A malicious user could otherwise edit the target size during the operation
to bypass their quota.

Admin metadata of volumes is not visible to clients, but Cinder supports
mapping select keys to the regular metadata, shadowing any user-set values of
the same key.

The key ``extend_new_size`` will be added to the list of visible admin
metadata in ``cinder/api/api_utils.py``, so that Nova is able to read the
target size of the extend operation.

OpenStack SDK
-------------

Support for the new volume action will be added to the OpenStack SDK, which
Nova will use to call it.

Nova
----

When the Nova API receives a ``volume-extended`` external server event, and
the call used the new microversion proposed in [4]_, it will check the target
compute service version.
If a target compute agent is too old to support the feature, the API will
discard the event and call the ``os-extend_volume_completion`` volume action
with ``"error": true``.

Otherwise, the event will be forwarded to the compute agent.
When handling the ``volume-extended`` external server event, compute will
check the volume status:

* If the volume status is ``extending``, then compute will attempt to read
  ``extend_new_size`` from the volume's metadata and use this value as the
  new size of the volume, instead of the volume size field.

  After successfully extending the volume, it will call the extend volume
  completion action of the volume, with ``"error": false``.

  If anything goes wrong, including ``extend_new_size`` being missing from the
  metadata, or being smaller than the current size of the volume, compute will
  log the error and call the extend volume completion action with
  ``"error": true``.

* For any other volume status the event will be handled as before.

The changes in Nova are detailed in the current version of the Nova spec at
[4]_.

os-reset_status
---------------

When resetting from status ``extending``, the ``os-reset_status`` volume
action will check for the ``extend_reservations`` key in the admin metadata.
If it finds quota reservation keys, it will try to roll them back.

This is done to avoid a pile up of quota reservations in case communication
between Cinder and Nova was lost and the status has to be reset to retry the
resize.

The keys ``extend_reservations`` and ``extend_new_size`` will then be removed
from the admin metadata.

Alternatives
------------

* A previous change tried to use the ``volume-extended`` external server event
  to support online extend for the NFS driver [1]_, but did not rely on
  feedback from Nova to Cinder at all.
  Instead, it would just set the new size of the volume, change the status
  back to ``in-use``, notify Nova, and hope for the best.

  If anything went wrong on Nova's side, this would still result in a volume
  state indicating that the operation was successful, which is not acceptable.

* The specs at [2]_ and [3]_ proposed a new synchronous API in Nova that can
  be used to trigger an assisted resize operation.
  This API would provide a single mechanism to trigger the resize operation,
  communicate the new size to Nova, and get feedback on the success of the
  operation.

  The problem with a synchronous API is, that RPC and API timeouts limit the
  maximum time an extend operation can take.
  For QEMU, this seemed to be acceptable, because storage preallocation is
  hard disabled for the ``block-resize`` command, and because all currently
  plausible file systems support sparse file operations.

  However, as reviewers in [2]_ have pointed out,  this may not be true for
  other volume or virt drivers that might require this API in the future.
  It would also break with the established pattern of asynchronous
  coordination between Nova and Cinder, which includes the assisted snapshot
  and volume migration features.

* Following this pattern, we could make the proposed API asynchronous and use
  a new callback in Cinder, similar to Nova's ``os-assisted-volume-snapshots``
  API, which uses the ``os-update_snapshot_status`` snapshot action to provide
  feedback to Cinder.

  The function of the new Nova API would then just be to trigger the operation
  and to communicate the new size.
  The question is then, whether that warrants adding a new API to Nova, since
  there are existing mechanisms that could be used for either.

* The existing mechanism for triggering the extend operation in Nova is, of
  course, the ``volume-extended`` external server event.
  Using it for this purpose, as this spec proposes, requires the target size
  to be transferred separately, because external server events only have a
  single text field that is freely usable, which for ``volume-extended``
  is already used for the volume ID.

  Besides storing it in the admin metadata, as this spec proposes, there is
  also the option of updating the size field of the volume, as [1]_ was
  essentially doing.

  This would require the volume size field to be reset on a failure.
  If an error response from Nova was lost, the volume would just keep the new
  size.
  We would need to extend ``os-reset_status`` to allow a size reset, or
  something similar to clean up volumes like this.
  This would be possible, but updating the size field only after the volume
  was successfully extended seems like a cleaner solution.

* We could also extend the external server event API to accept additional data
  for events, and use this to communicate the new size to Nova.

  This option was judged favorably by reviewers on the previous version of
  this spec, [2]_, but it would be a more complex change to the Nova API.

  However, if additional data fields become available in a future version of
  the external server event API, it would be a relatively minor change to use
  those instead of the volume metadata.

Data model impact
-----------------

None

REST API impact
---------------

Starting with the new microversion, the
``POST /v3/{project_id}/volumes/{volume_id}/action`` API will accept request
bodies of the following form:

.. code-block:: json

    {
        "os-extend_volume_completion": {
            "error": false
        }
    }

with ``error`` indicating success or failure of the resize operation.

If the volume does not exist, the return code will be ``404 Not Found``.

If the volume status and admin metadata do not indicate that Cinder was
waiting for an extend volume completion action, the return code will be
``400 Bad Request``.

Otherwise the return code will be ``202 Accepted``.

The new volume action is intended to only be used by Nova and will require
the caller to have admin permissions.

Security impact
---------------

None

Active/Active HA impact
-----------------------

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
  kgube

Work Items
----------

* Move extend completion code from ``VolumeManager.extend_volume`` to new
  method and add tests.
* Create new volume action and add unit tests.
* Add a new microversion for the new ``os-extend_volume_completion`` action.
* Add OpenStack SDK support.
* Add Nova support.
* Update drivers to use the feature.
* Adapt the ``devstack-plugin-nfs-tempest`` CI-jobs to also test online volume
  extend.

Dependencies
============

* Nova support of the callback [4]_.

Testing
=======

* Unit tests for the volume action will test the conditions all possible API
  responses.
* Unit tests for ``VolumeManager.extend_volume`` will test all the code paths
  described in `VolumeManager.extend_volume`_.
* The new volume action cannot be independently tested by Tempest, because it
  requires the volume to be in a state that cannot be reproduced externally.
  It is, however, covered by the existing tests for online volume extend when
  they are run with one of the volume drivers that use this feature.
  The ``devstack-plugin-nfs-tempest`` jobs that run as part of the Cinder and
  Nova CI gates will be configured to enable online volume extend tests.

Documentation Impact
====================

The Block Storage API reference will be updated to include the new volume
action.

The volume driver support matrix will be updated to show online resize support
for the affected drivers.

References
==========

.. [1] https://review.opendev.org/c/openstack/cinder/+/739079
.. [2] https://review.opendev.org/c/openstack/nova-specs/+/855490/6
.. [3] https://review.opendev.org/c/openstack/cinder-specs/+/864020
.. [4] https://review.opendev.org/c/openstack/nova-specs/+/917133

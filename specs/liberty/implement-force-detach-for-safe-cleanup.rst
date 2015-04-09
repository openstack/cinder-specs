..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Implement force_detach for safe cleanup
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/implement-force-detach-for-safe-cleanup

Currently, there is no API to synchronize the state in the Cinder database and
the backend storage when a volume is in a stuck 'attaching' or 'detaching'
state. For example, if a volume is stuck in detaching, but is
attached/connected by the storage driver, the admin is allowed to use
'cinder reset-state <vol_id>' to set the state of the volume to 'detached'.
This will be mismatched with the backend storage state, which has the volume
connected/exported to the compute host and nova instance. If the volume state
is reset to 'available' and 'detached', it can be attached to a second
instance (note that this is not multi-attach) and lead to data corruption.
This spec proposes to plumb cinder APIs that allow an admin to set the state to
 'available' and 'detached' in a safe way, meaning that the backend storage and
 cinder db are synchronized.
An attempt was made to merge a Nova spec to add a '--force' option, but it has
stalled and will likely be replaced with some Nova code changes. See:
https://review.openstack.org/#/c/84048/44
Nova devs have proposed an idempotent 'nova volume-detach...' that will
1) detach the volume from the VM (libvirt.detach) 2) call cinder to detach
and Ignore Cinder Errors 3) remove the BlockDeviceMapping from Nova DB.
This will have the effect of cleaning up Nova side but possibly leaving
Cinder in an error/out_of_sync state, and will require a manual cleanup for
Cinder, i.e. cinderclient 'force-detach'.
(cinderclient patch: https://review.openstack.org/#/c/187305/)

Problem description
===================

It is not uncommon to have a volume stuck in a state that prevents the user
from using the volume or changing state, i.e. 'attaching', 'deleting',
'creating', 'detaching'. The python-cinderclient has the command 'reset-state'
to allow an admin to change the volume state in the cinder database to any
state desired.
In order to fix the problem in a safe and robust way, however, the admin must
check the state of the volume with regards to the backend storage and Nova. Is
the volume attached according to the metadata of the backend? Is there an iSCSI
connection (or FC) to the compute host? Does Nova BlockDeviceMapping have an
entry for the volume? Cleanup to the Cinder DB is possible, cleanup on the
storage backend may be possible, and changing the Nova DB requires direct
changes (SQL commands) on the Nova database. Synchronizing state requires
detailed knowledge of all of the components, and often invasive and dangerous
changes to production systems.
Changing python-cinderclient 'reset-state' has been proposed[1], but that
idea was rejected in favor of implementing separate APIs for achieving the
desired state[2]. This spec is to implement 'force_detach'. There is already a
'force_delete' API for help with volumes stuck in 'creating' or 'deleting'.
In fact, there is already a force_detach API that calls terminate_connection
and detach, but this appears to be unused elsewhere in the code base and
should be properly implemented to test whether the driver can comply, and in
the case of success from the driver the function should cleanup the Cinder DB.
It could be argued that there should be an analogous 'force_attach' or perhaps
more aptly named 'force_rollforward_attach', but it seems that simply forcing
a detach of a volume will put the volume back in a state where the attach can
be attempted again.

UseCase1: Cinder DB 'attaching', storage back end 'available', Nova DB
does not show block device for this volume.
An attempt was made to attach a volume using 'nova volume-attach <instance>
<volume_id>'
The Cinder DB is set to 'attaching', but the volume was never attached.
Current Fix: Using python-cinderclient 'reset-state' to set the Cinder DB to
'available' will fix this.
Proposed Fix: An attempt to detach from Nova will fail, since Nova does not
know about this volume.
Implementing Cinder force_detach and exposing to the cinderclient will allow
 cinder to call the backend to cleanup (default implementation is
terminate_connection and detach, but can be overridden) and then set Cinder DB
 state to 'available' and 'detached'.

UseCase2: Cinder DB 'attaching', storage back end 'attached', Nova DB
shows block device for this volume.
The Cinder DB is set to 'attaching', but the volume was attached.
Current Fix: Using python-cinderclient 'reset-state' to set the Cinder DB to
'available' will result in an out-of-sync state, since the volume is, in fact,
attached. Nova will not allow this volume to be re-attached.
Proposed Fix: An attempt to detach from Nova will cleanup on the Nova side
(after proposed Nova changes) but fail in Cinder since the state is
 'attaching'.
Implementing Cinder force_detach and exposing to the cinderclient will allow
 cinder to call the backend to cleanup (default implementation is
terminate_connection and detach, but can be overridden) and then set Cinder DB
 state to 'available' and 'detached'.

UseCase3: Cinder DB 'detaching', storage back end 'available', Nova DB
does not show block device for this volume.
An attempt was made to detach a volume using 'nova volume-detach <instance>
<volume_id>'
The Cinder DB is set to 'detaching' and the volume is actually detached.
Current Fix: Using python-cinderclient 'reset-state' to set the Cinder DB to
'available' will fix this.
Proposed Fix: An attempt to detach from Nova will fail, since Nova does not
know about this volume.
Implementing Cinder force_detach and exposing to the cinderclient will allow
 cinder to call the backend to cleanup (default implementation is
terminate_connection and detach, but can be overridden) and then set Cinder DB
 state to 'available' and 'detached'.

UseCase4: Cinder DB 'detaching', storage back end 'attached', Nova DB
has a block device for this volume.
An attempt was made to detach a volume using 'nova volume-detach <instance>
<volume_id>'
The Cinder DB is set to 'detaching', but the volume is actually attached.
Current Fix: Using python-cinderclient 'reset-state' to set the Cinder DB to
'available' will result in an out-of-sync state, since the volume is, in fact,
attached. Nova will not allow this volume to be re-attached.
Proposed Fix: An attempt to detach from Nova will cleanup on the Nova side
(after proposed Nova changes) but fail in Cinder since the state is
 'attaching'.
Implementing Cinder force_detach and exposing to the cinderclient will allow
 cinder to call the backend to cleanup (default implementation is
terminate_connection and detach, but can be overridden) and then set Cinder DB
 state to 'available' and 'detached'.

UseCase5: During an attach, initialize_connection() times out. Cinder DB is
'available', volume is attached, Nova DB does not show the block device.
Current Fix: None via reset-state. Manual intervention on the back end
storge is required
Proposed Fix: Code in nova/volume/cinder.py L#366 calls initialize_connection,
which can time out. A patch is up for review to put this in a try block and
cleanup if there is an exception:
https://review.openstack.org/#/c/138664/6/nova/volume/cinder.py
This patch could be modified to call force_detach() instead of
terminate_connection to insure the DB status entry is set to available
and allow for driver detach and any driver specific code to be called for
cleanup.

Proposed change
===============
Nova must be changed, since it currently checks the state in the Cinder DB
and will fail a call to 'volume-detach' if Cinder does not show the volume
to be 'attached' and 'in-use'. It is being proposed that Nova ingnore Cinder
state and do the libvirt.detach and removal of volume entry in BDM. Nova will
call Cinder and ignore any errors, leaving cleanup on the Cinder side to be
accomplished via manual intervention (i.e. 'cinder force-detach....'
(Links to proposed Nova changes will be provided ASAP)

Cinder force-detach API currently calls:
    volume_api.terminate_connection(...)
    self.volume_api.detach(...)
This will be modified to call into the VolumeManager with a new force_detach(...)

api/contrib/volume_actions.py: force_detach(...)
    try:
        volume_rpcapi.force_detach(...)
    except: #catch and add debug message
        raise #something

    self._reset_status(..) #fix DB if backend cleanup is sucessful

volume/manager.py: force_detach(...)
   self.driver.force_detach(..)

Individual drivers will implement force_detach as needed by the driver, most
likely calling terminate_connection(..) and possibly other cleanup. The
force_detach(..) api should be idempotent: It should succeed if the volume is
not attached, and succeed if the volume starts off connected and can be
sucessfuly detached.

Alternatives
------------

Leave things as they are, requiring the admin to make manual changes using APIs
or commands on the back end storage to keep the state in sync. Nova has no API
to cleanup the BlockDeviceMapping table. Using 'reset-state' can work, as in
UseCase1 and UseCase3, or it can break things and render a volume incapable
of being attached, as in UseCase2 and UseCase4.

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

Detach notification will indicate force_detach

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

BaseVD class will implement force_detach as it is today, calling
terminate_connection and detach. Driver developers can override this
function if there is more they wish to do in their driver.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
scott-dangelo

Work Items
----------

Changes to core Cinder code
Drivers implementation of force_detach, if it is desired to override
cinderclient changes (https://review.openstack.org/#/c/187305/)

Dependencies
============

None
Nova changes are independent of this spec

Testing
=======

Unit tests for test_force_detach* will be modified as appropriate.
Tempest tests will be added to verify a volume in an attaching or detaching
state can be force_detached and then successfuly re-attached.


Documentation Impact
====================

Docs will need to be updated for the python-novaclient changes.


References
==========

[1] https://blueprints.launchpad.net/cinder/+spec/reset-state-with-driver
    https://review.openstack.org/#/c/134366/2

[2] https://etherpad.openstack.org/p/cinder-meetup-winter-2015 L#405


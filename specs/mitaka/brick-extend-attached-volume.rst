..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Add API to os-brick to allow extending an attached volume
=========================================================

https://blueprints.launchpad.net/cinder/+spec/brick-extend-attached-volume

Add extend_volume API to os-brick connector objects to allow
updating the host kernel size information for an attached volume.


Problem description
===================

Currently, Cinder has the ability to extend volumes that aren't attached.
Unfortunately, this doens't work for attached volumes, due to the host
kernel that the volume is attached to, won't see the new size unless actions
are taken on the host.  The linux kernel doesn't automatically see the size
changing on the remote storage server.  This spec outlines the idea of adding
those required actions on the host server when an attached volume is extended
in size.


Use Cases
=========

An OpenStack user has a Cinder volume attached to a host/VM and they want to
extend/grow the volume while it's attached.

Proposed change
===============

Add a new os-brick Connector api called "extend_volume" that does the work
on the host to get the volume size updated.  This will also have to take into
account the raw device and the multipath device, if available.  It will return
the size of the volume after the resize is complete or None if it failed.

This spec doesn't cover the work needed by Cinder and Nova to make the solution
complete.   This spec simply covers the work needed by os-brick, to ensure the
volume that's attached to the nova compute host can update it's host kernel
to see the volume's new size.  Cinder will have to change, to allow attached
volumes to be extended, and when complete call a new Nova API to initiate the
work on the compute host.  Then Nova will have to issue a blockresize in virt
to get the VM's virtual device resized.  For the volume types such as RBD,
where Nova doesn't use os-brick as the library for managing the volume, Nova
will have to do the work of issuing the commands to extend the volume itself.

Alternatives
------------

Unfortunately, if a volume is attached to a VM, there is nothing an end user
can do, because the Nova compute host kernel has to see the changes before the
VM can see the new size.

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

The end user, that owns the VM will may have to issue a filesystem resize to
get the native filesystem on the volume to recognize the new volume size.  For
ext* style filesystems, users can use resize2fs.

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
  walter-boring

Work Items
----------

Add the new API to the base Connector class and then add the common required
code to the linuxscsi.py to do the real work.


Dependencies
============

* In order for this to be used by the OpenStack end user, Cinder will have to
  optionally allow for attached volumes to be extended.

* There will also need to be some Nova work done to initiate the call into
  os-brick's new API to do the work, and then notify the VM after it's
  successful.

Testing
=======

Until the end to end Cinder -> Nova capability is in place, there isn't a way
to automate the testing of this.  For now, it's a manual process.

Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

A related cinder-spec that has been up for a while here:

* https://review.openstack.org/#/c/200627/

Related information on how to manually do the work of notifying the host
kernel:

* https://help.ubuntu.com/lts/serverguide/multipath-admin-and-troubleshooting.html

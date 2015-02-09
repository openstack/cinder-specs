..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Attach a single volume to multiple hosts
==========================================

https://blueprints.launchpad.net/cinder/+spec/multi-attach-volume

Currently, Cinder only allows a volume to be attached to a single
host or instance.  There are times when a user may want to be able
to attach the same volume to multiple instances.

Problem description
===================

Currently, Cinder only allows a volume to be attached to one instance
and or host at a time.  Nova makes an assumption in a number of places
that assumes the limitation of a single volume to a single instance.

* cinderclient only has volume as a parameter to the detach() call.  This
  makes the assumption that a volume is only attached once.

* nova assumes that if a volume is attached, it can't be attached again.
  see nova/volume/cinder.py: check_attach()

Use Cases
---------
Allow users to share volumes between multiple guests using either
read-write or read-only attachments. Clustered applications
with two nodes where one is active and one is passive. Both
require access to the same volume although only one accesses
actively. When the active one goes down, the passive one can take
over quickly and has access to the data.

Proposed change
===============

To enable the ability to attach a volume to multiple hosts or instances,
first we need a new volume_attachment table that tracks each attachment
for cinder volumes.   We will migrate the existing columns from the volume
table (attached_host, instance_uuid, mountpoint, attach_time, attach_mode).
The volume_attachment table will have an id (known as the attachment_id) that
tracks individual attachment records for each volume.  The existing cinder
volume object that is returned in the API calls has a list of attachments.
This attachment list will now also contain the attachment_id for each of the
attachments.

That attachment_id is what nova will pass into cinder during detach API calls,
so that cinder knows which attachment to detach.   The cinder API will support
a default attachment_id of None, which will try to do a detach if there is only
a single attachment.

If cinder doesn't get the attachment_id, and the volume only has 1 attachment,
then the detach will work.  If the volume has more than one attachment, then
cinder will return an error saying that the volume has more than one attachment
and needs the attachment_id.

The volume table will be updated to include a new column called 'multiattach',
which is a boolean flag to signal cinder that the volume can/can't be attached
more than once.  The cinder API and cinderclient will be updated to support
setting the multiattach flag at volume attach time.  We can add support to
updating a volume record to enable the multiattach flag as well, or as a
follow up patch.



Alternatives
------------

The only alternative is for a user to clone a volume and attach the clone
to the second instance.   The downside to this is that any changes to the
original volume don't show up in the mounted clone.


Data model impact
-----------------

The volume table will be modified to remove certain columns as they will be
migrated to the new volume_attachments table.

A new volume_attachments table will be created to track all of the attachments
for volumes.

Any existing attachments will be migrated to the new schema.


REST API impact
---------------

The REST API will be changed in 2 ways.

1) A new multiattach flag can be passed for the create volume action.

2) The existing REST API already returns a list for attachments, even though
   only one attachment is supported.  Now if a volume is multiattach enabled,
   the attachments can return more than 1 entry.

3) Detach will accept an optional attachment_id to specify which attachment
   to try and remove.   If the volume is attached to more than once host or
   instance and the attachment_id is not passed in, the API will raise an
   exception.


Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------

When listing volumes, the multiattach flag will be shown from the command line
client.

Also, the command line client will include an optional --allow-multiattach
to cinder create.  By default multiattach is False

Performance Impact
------------------

A possible performance hit is the extra fetches to that volume_attachment
table at volume fetch time.  This is a simple foreign key table join, which
is indexed.

Other deployer impact
---------------------

None

Developer impact
----------------

Volume driver developers should ensure that they have their volumes attached
to more than one instance.   We should do a follow up patch on drivers to
add a 'multiattach' capability being reported.   We could update the
scheduler to filter out backends that can't do multiattach.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  walter-boring (walter.boring@hp.com)


Work Items
----------

* Cinder needs to be updated to support the new API changes and to support
  exporting a volume more than once.

* Cinderclient needs to be updated to support the new multiattach flag at
  create time.

* Cinderclient needs to be updated to support the new attachment_id being
  passed at detach volume.

Dependencies
============

* The nova-spec that is needed to support this, has already been approved.
  https://github.com/openstack/nova-specs/blob/master/specs/kilo/approved/multi-attach-volume.rst

Testing
=======

There will need to be new tempest tests in place to gate on multiattach.
That work is going on now as well.
https://review.openstack.org/#/c/153038/

Documentation Impact
====================

Documentation should be updated to reflect the new API changes as well as the
new --allow-multiattach flag at volume create time.


References
==========

* Blueprints for all affected projects
  https://blueprints.launchpad.net/openstack/?searchtext=multi-attach-volume

* Nova tests changes:
  https://review.openstack.org/#/c/153038/

* Cinder wiki page:
  https://wiki.openstack.org/wiki/Cinder/blueprints/multi-attach-volume

* Horizon work:
  https://blueprints.launchpad.net/horizon/+spec/cinder-multi-attach-volume

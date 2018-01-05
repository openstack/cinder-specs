..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Enable multiattach of volumes
=============================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/multi-attach-v3-attach

The ability to attach a volume to multiple hosts/servers simultaneously
is a use case that has been asked for over the years.  The implementation
of the Attach workflow in Cinder however made this somewhat difficult and
less than robust.

With the new Cinder Attachment API's [1]_ this is
a bit easier because Attachments are treated as independent objects.

This spec proposes a set of changes to enable and control the use of
multiattach volumes from the Cinder side.

Problem description
===================

Currently we refuse to attach a volume that is already attached (in-use).
There are situations where a cloud user may have deployed a clustered
file-system on a Cinder volume and they would like the ability to attach
a volume to multiple hosts/servers at the same time.

Note that Cinder is not providing any sort of Filesystem checks or sharing
overlays on the volume, it's completely up to the user to handle this and
determine if they can in fact attach a volume to multiple locations without
incurring filesystem/data corruption.

The problem that we are proposing to solve with this spec is simply to enable
the ability to attach a volume to multiple locations, and provide the cloud
operator with policies to enable/disable the functionality for the cloud.  We
do NOT offer any mechanism to keep a user from corrupting their data or doing
something terrible.

Terminology
-----------

Multi-Attach RO:
  The ability to attach a volume as one would normally do (Read Write access)
  and then to attach that volume to another host(s)/server(s) in Read Only
  mode.  Note, this is actually tricky because this typically will require that
  the consumer (ie Nova/KVM) knows how to set and enforce Read Only on an
  attached volume.

Multi-Attach RW:
  The ability to attach a volume as one would normally do (Read Write access)
  and then to attach that volume to another host(s)/server(s).  In this case
  there is no differentiation between the initial attachment and any subsequent
  attachments.  They are all treated as independent items and are all Read
  Write volumes with no real distinction beyond what's provided with a single
  attachment.


Multi-Attach Capable Backend:
  Some backend devices just flat out may not support this.  Those that do are
  considered to be "Multi-Attach Capable" backend devices and need to report
  this in their capabilities.

Example Volume Types to communicate Multi Path info:
  Read only Multi Attach type (extra-specs)
  {
  'multiattach': '<is True>',
  'mode': 'RO'
  }

  Read/Write Multi Attach type (extra-specs)
  # Default is RW so omission of mode in extra-specs == RW
  {
  'multiattach': '<is True>',
  }

  # Also accept explicit request if so desired
  {
  'multiattach': '<is True>',
  'mode': 'RW'
  }

  It will be up to the Cinder scheduler to determine if the policy allows these
  types of volumes to be created, and it will then be up to the driver to form
  the corresponding Connection and Volume attributes appropriately in it's
  model returns.

  Again NOTE we will NOT allow retype of attachment setting for an `in-use`
  volume.

Use Cases
=========

* There are products that can be used like Oracle RAC that
  would make things like H/A databases via a shared Block device possible which
  is something of interest.
* The other use case that's been proposed is the
  ability to have things like passive stand-by servers/volumes.
* Analyzing/Fleecing an Instance from another Instance or host

Proposed change
===============

MultiAttach policy
------------------

Create a Cinder policy that enables/disables the ability to multi-attach
a volume.  This policy is a global policy for the endpoint and simply either
enables or disables the capability.  There's no need for any granularity around
RO and RW policies, mostly as these can't be enforced from the Cinder side
anyway.  Things like RO and RW will have to be handled by the consuming service
(Nova, Ironic etc).

MultiAttach capable volume type
-------------------------------

In order to ensure that a volume is created on a backend that supports multiple
attachments, it will need to be created on a backend that allows this.  The
only way to safely control this is be requiring that a type of
"multiattach" is created and used on volume creation.

If an additional attach request is made to a volume that is NOT of the type
multi-attach enabled the request should fail as the volume is already in-use,
and either a new volume should be created of the correct type, or the volume
should be retyped.  Of course if the policy allows and the type is correct the
additional attachment request should be processed and the existing
attachment(s) are left active and NOT interrupted.

To avoid various corner cases and confusion for users in what's allowed and
what's not, this spec proposes we set a hard-code rule that disallows the
retyping of attachment parameters of an `in-use` volume.  Regardless of what
those changes are; multi-->non-multi, non-multi-->multi, RO, RW etc etc.

Any retype command should inspect the multiattach keys and if there is any
change being requested in said keys the retype should fail immediately at the
API service.

NOTE that the default will be multiattach:False

There is an additional required task, the `--allow-multiattach`
parameter included with volume create needs to be deprecated and it needs to be
very explicit that that flag is NOT relevant to the implementation of
multi-attach that is being proposed in this spec and implemented in Cinder API
microversion xyz.

Special considerations for force-detach; recent changes have been added that
treat things like detach of attachments with no connector as `force detach`
operations.  This should be fine in a multi-attach scenario because the force
detach is ONLY issued against that instantiation of an attachment.  In other
words the only thing force detach does is ignore state checks and disconnects
the specified session, leaving others intact.  The process on the Cinder side
should follow the standard multi-attach detach process with the exception of
ignoring the status of the volume.

FWIW that call rarely works as users expect anyway, so maybe this will be
a good time to revisit it.  Perhaps deprecate it, and include a force parameter
to the attachment-delete API call itself?


Alternatives
------------

* Be cloudy and write clustered apps
* Use things like independent data instances
* Get over it, you probably don't really need this

Data model impact
-----------------

N/A
All of the needed changes to the data model should already be in place.
Inparticular the existing `multiattach` column on the volume object, which
will signal the consumer that they're working with a multiattach capable
volume.  So, if a volume of type multiattach:True is requested/created,
it's multiattach column is set to True, else False.

To elaborate on the create flow:
Admin creates a volume type `multiattach` with extra-specs:
{
'multiattach': '<is True>'
}

If a user desires a multiattach volume, he/she issues a create call specifying
the multiattach type:
`cinder create --volume-type multiattach --name my-mavol 20`

If there are NO backends reporting `multiattach: <is True>` then scheduling
will fail.  Unfortunatly this process includes the task-flow retries and the
object will be created, the caller will get a 202 response, but after taskflow
and the scheduler finish retries and fail to find a backend the volume status
will be set to error.  It may be possible to speed this up if we need to and
put capability checks into the API layer so that we could respond immediately
with an Invalid Request. We do provide an API call now that pulls capabilities
from the system, so we could make that check upon receipt of the create request
and make sure it's possible to fulfill the request.  It would probably be wise
to cache this info based on the periodic capability reporting instead of
fetching it each time.

REST API impact
---------------

This has a number of impacts on the API, the most obvious of which is the fact
that you can attach a volume multiple times.  The other changes that may not be
so obvious are things like representing a volume's attachment status, and
managing state changes when a secondary attachment is processed or removed.

The current representation of volume-status is likely to be insufficent once
multiple attachments are enabled.

Security impact
---------------

N/A

Notifications impact
--------------------

N/A

Other end user impact
---------------------

User can potentially attach a volume to multiple servers, and corrupt their
data.

Performance Impact
------------------

N/A

Developer impact
----------------

Drivers will need to add a capabilities field "multiattach: True/False", and
do any special handling on their end for connecting/disconnecting volumes in
this category.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  None

Other contributors:
  None

Work Items
----------

* Update Cinder attach/detach to accomodate shared targets
  (most of this is being done independently, see `Dependencies`
  section for more info)
* Implement policy changes
* Implement changes to API to allow/ignore existing attachments on
  attachment-create calls.
* Update the attach/detach volume-status transitions to be multiattach aware
  and make sure they reflect the correct values.
* Update the detail volume view and possibly the summary view to clearly inform
  the user when a volume is attached to multiple servers.

Dependencies
============

* Handling of disconnects for devices using shared targets
  Initial work is under review here:  https://review.openstack.org/#/c/520676/
  Additionally, the API will need a micro version bump and additions to the
  response views for volumes.
* Required patches for service-uuid have already merged
  https://review.openstack.org/#/c/519025/


Testing
=======

https://review.openstack.org/#/c/266605/

New unit tests will be added to test the changed code and functional testing
will need to be added as well.

Documentation Impact
====================

This will require updates to both deployment guides as well as end-user guides.

References
==========

.. [1] https://specs.openstack.org/openstack/cinder-specs/specs/ocata/add-new-attach-apis.html

Related Nova spec:
https://specs.openstack.org/openstack/nova-specs/specs/queens/approved/cinder-volume-multi-attach.html

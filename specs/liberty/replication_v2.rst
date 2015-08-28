..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VolumeReplication_V2
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/replication-v2

This spec proposes Version 2 of replication.  The goal is to take
some of the lessons we've learned from the first version of
replication that was added in teh Juno release and see if we can
improve it a bit and make it something that is more widely usable
by other backend devices.

This spec proposes a use of our entire tool-box to implement replication.

1. Capabilities - determine if we can even do anything related to replication

2. Types/Extra-Specs - provide mechanism for vendor-unique custom info and
   help level out some of the unique aspects among the different back-ends.

3. API Calls - provide some general API calls for things like enable,
   disable etc

It would also be preferrable to simplify the state management a bit if we can.


Problem description
===================
The current implementation is fairly complex and  has proven to be difficult
to implement for backend devices as well as difficult to maintain.  There
is also some concern around how states are managed and stored in the data
base.

The existing design is great for some backends, but is challenging for many
devices to fit in to.

Use Cases
=========
TBD.

Proposed change
===============

This spec proposes that we make some fairly significant changes to the
replication feature in Cinder.  We'd still rely on using capabilities and
types to identify whether a backend supports replication and ensure we
can place a volume correctly if we want to use the feature.  The big
difference however is around the creation of the replica and the life-cycle
that goes with it.  For the first iteration we would only support a single
remote device, but this is something that's considered in this spec and
could easily be extended to be included in future work. Backend devices
(drivers) would be listed in the cinder.conf file and we add entries
to indicate pairing.  This could look something like this in the conf file:

    [driver-foo]
    volume_driver=xxxx
    valid_replication_devices='backend=backend_name-a',
      'backend=backend_name-b'....

Alternatively the replication target can potentially be a device unknown
to Cinder

    [driver-foo]
    volume_driver=xxxx
    valid_replication_devices='remote_device={'some unique access meta}',...

Or a combination of the two even

    [driver-foo]
    volume_driver=xxxx
    valid_replication_devices='remote_device={'some unique access meta}',
      'backend=backend_name-b'....

NOTE That the remote_device access would have to be handled via the
configured driver.

This proposal suggests that we would decouple the replication information
from the create call.  The flow would be something like this:

* Create a volume of type "replication_capable=True" and any custom info needed
  by a specific backend.

* Cinder uses the existing functionality of the capabilities filter to pick
  a backend that supports the requested replication type.  This is consistent
  with the current create workflow, we just add another capability to the
  scheduler.

* Add the following API calls
  enable_replication(volume)
  disable_replication(volume)
  failover_replicated_volume(volume)
  udpate_replication_targets()
    [mechanism to add tgts external to the conf file * optional]
  get_replication_targets()
    [mechanism for an admin to query what a backend has configured]


Special considerations
-----------------
* volume-types
  There should not be a requirement of an exact match of volume-types between
  the primary and secondary volumes in the replication set.  If a backend "can"
  match these exactly, then that's fine, if they can't, that's ok as well.

  Ideally, if the volume fails over the type specifications would match, but if
  this isn't possible it's probably acceptable, and if it needs to be handled
  by the driver via a retype/modification after the failover, that's fine as
  well.

* async vs sync
  This spec assumes async replication only for now.  It can easily be
  extended later for the synchronous case, but for now it's specific
  to async.  If/When sync is added it can be specified as an additional
  backend capability.  It's also possible for this to be specified via
  extra-specs if desired.

* transport
  Implementation details and the *how* the backend performs replication
  is completely up to the backend.  The requirements are that the interfaces
  and end results are consistent.

* Cinder does not need to be aware of both backend devices but CAN be
  This spec is intended to provide flexibility, that means that if an
  admin wishes to configure a backend device that is unknown to Cinder
  that absolutley fine.  The opposite is true as well of course, that
  detail is outlined in this spec.

* Tenant visibility
  The visibility by tenants is LIMITED!!!  In other words the tenant
  should know very little about what's going on (if anything at all).

  For example, a service provider may sell replication simply as a
  volume-type defined as "highly available" and have that equate to
  replication.  The point is there's absolutely no reason an end user
  should have to know anything at all about replication (unless it costs
  them more money).

* What about devices that can't do individual volume-rep
  It's up to them to figure out what they want to do.  If for example
  they replicate by pool, then maybe they can be sophisticated enough to
  put all the volumes of replication type in the same pool and replicate
  the entire pool.

  There are lots of options here I think, the point of this spec is that
  it does not exclude any implementation.

Workflow diagram
-----------------
Create call on the left:
  No change to workflow

Replication calls on the right:
  Direct to manager then driver via host entry

      +-----------+
 +--< +Volume API + >---------+        Enable routing directly to
 |    +-----------+           |        Manager then driver, via host
 |                            |
 |                            |
 |    +-----------+           |
 +--> + TaskFlow  |           |
 +--< +-----------+           |
 |                            |
 |                            |
 |    +-----------+           |
 +--> + Scheduler |           |
 +--< +-----------+           |
 |                            |
 |                            |
 |    +-----------+           |
 +--> +  Manager  | <---------+
 +--< +-----------+ >---------+
 |                            |
 |                            |
 |    +-----+-----+           |
 +--> +  Driver   + <---------+
      +-----+-----+

In the case of calls like attach, extend, clone, delete etc;
if either the backend host is not reachable, or if the primary_host_status
column is set, we'll redirect to the host in the secondary_hosts
column.  If that's unavailable then we fail, just like we do today.

See DB section below

Alternatives
------------

There are all sorts of alternatives, the most obvious of which is to leave
the implementation we have and iron it out.  Maybe that's good, maybe that's
not.  In my opinion this approach is simpler, easier to maintain and more
flexible; otherwise I wouldn't propose it.  The fact that there's only
one vendor that's implemented replication in the existing setup and they
have a number of open issues currently we're not causing a terrible amount
of churn or disturbance if we move forward with this now.

The result will be something that should be easier to implement and as an
option will have less impact on the core code.


Data model impact
-----------------

* What new data objects and/or database schema changes is this going to
  require?

None, for the first pass we should be able to effectively use the existing
replication related columns.

REST API impact
---------------

We would need to add the API calls mentioned above:
  enable_replication(volume)
  disable_replication(volume)
  failover_replicated_volume(volume)
  udpate_replication_targets()
    [mechanism to add tgts external to the conf file * optional]
  get_replication_targets()
    [mechanism for an admin to query what a backend has configured]

I think augmenting the existing calls is better than reusing them, but we can
look at that more closely in the submission stage.

Security impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

  Nope

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

  Nope, not that I know of

* Does this change involve cryptography or hashing?

  Nope, not that I know of

* Does this change require the use of sudo or any elevated privileges?

  Nope, not that I know of

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

  Nope, not that I know of

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  Nope, not that I know of

For more detailed guidance, please see the OpenStack Security Guidelines as
a reference (https://wiki.openstack.org/wiki/Security/Guidelines).  These
guidelines are a work in progress and are designed to help you identify
security best practices.  For further information, feel free to reach out
to the OpenStack Security Group at openstack-security@lists.openstack.org.

Notifications impact
--------------------

Please specify any changes to notifications. Be that an extra notification,
changes to an existing notification, or removing a notification.

Other end user impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* Does this change have an impact on python-cinderclient? What does the user
  interface there look like?

Performance Impact
------------------

Describe any potential performance impact on the system, for example
how often will new code be called, and is there a major change to the calling
pattern of existing code.

Examples of things to consider here include:

* A periodic task might look like a small addition but when considering
  large scale deployments the proposed call may in fact be performed on
  hundreds of nodes.

* Scheduler filters get called once per host for every volume being created,
  so any latency they introduce is linear with the size of the system.

* A small change in a utility function or a commonly used decorator can have a
  large impacts on performance.

* Calls which result in a database queries can have a profound impact on
  performance, especially in critical sections of code.

* Will the change include any locking, and if so what considerations are there
  on holding the lock?

Other deployer impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other volume drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

* If this change is a new binary, how would it be deployed?

* Please state anything that those doing continuous deployment, or those
  upgrading from the previous release, need to be aware of. Also describe
  any plans to deprecate configuration values or features.  For example, if we
  change the directory name that targets (LVM) are stored in, how do we handle
  any used directories created before the change landed?  Do we move them?  Do
  we have a special case in the code? Do we assume that the operator will
  recreate all the volumes in their cloud?

Developer impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the driver API, discussion of how
  other volume drivers would implement the feature is required.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  john-griffith

Other contributors:
  <launchpad-id or None>

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* Include specific references to specs and/or blueprints in cinder, or in other
  projects, that this one either depends on or is related to.

* If this requires functionality of another project that is not currently used
  by Cinder (such as the glance v2 API when we previously only required v1),
  document that fact.

* Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?


Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.

Is this untestable in gate given current limitations (specific hardware /
software configurations available)? If so, are there mitigation plans (3rd
party testing, gate enhancements, etc).


Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.

Obviously this is going to need docs


References
==========

Please add any useful references here. You are not required to have any
reference. Moreover, this specification should still make sense when your
references are unavailable. Examples of what you could include are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related specifications as appropriate (e.g. link to any vendor documentation)

* Anything else you feel it is worthwhile to refer to

  The specs process is a bit much, we should revisit it.  It's rather
  bloated, and while the first few sections are fantastic for requiring
  thought and planning, towards the end it just gets silly.

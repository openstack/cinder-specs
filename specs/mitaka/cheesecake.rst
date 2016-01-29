..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cheesecake
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/replication

This spec proposes further refinement to the Cinder replication.
After more vendors have tried to implement replication and we've
learned more lessons about the differences in backends and their
semantics we've decided we should step back and look at simplifying
this even further.

The goal of the new design is to address a large amount of confusion
and differences in interpretation.  Rather than try and cover multiple
use cases in the first iteration, this spec aims to address a single
fairly well defined use case.  Then we can iterate and move on from
there.

Problem description
===================
The existing design is great for some backends, but is challenging for many
devices to fit in to.  It's also filled with pitfalls with the question of
managed/unmanaged, not to mention trying to deal with failing over some
volumes and leaving others. The concept of failing over on a volume basis
instead of on a device basis while nice for testing doesn't fit well into
the intended use case and results in quite a bit of complexity and also is not
something that a number of backends can even support.

Use Cases
=========
This is intended to be a DR mechanism.  The model Use Case is a catastrophic
event occurring on the backend storage device, but some or all volumes that
were on the primary backend may have been replicated to another backend device
in which case those volumes may still be accessible.

The flow of events are as follows:
1.  Admin configures a backend device to enable replication.  We have a
configured cinder backend just as always (Backend-A) but we add config
options for a replication target (Backend-B).

    a.  We no longer deal with differentiation between managed and unmanaged
        Now, to enable a replication target(s), the replication_target entry
        is the ONLY method allowed and is specified as a section in the driver.

    b.  Depending on the back-end device enabling this may mean that EVERY
        volume created on the device is replicated, or for those that have
        the capability and if admins choose to do so a Volume-Type of
        "replicated=True" can be created and used by tenants.

    Note that if the backend only supports replicating "all" volumes, or
    if the Admin wants to set things up so that "all" volumes are
    replicated that the Type creation may or may not be necessary.

2.  Tenant creates a Volume that is replicated (either by specifying
    appropriate Type, or by the nature of the backend device)
    Result in this example is a Volume we'll call "Foo"

3.  Backend-A is caught in the crossfire of a water balloon fight that
    shouldn't have been taking place in the data center, and looses it's magic
    smoke, "It's dead Jim!"

4.  Admin issues "cinder replication-failover" command with possible arguments
    a.  Call propagates to Cinder Driver, which performs appropriate steps for
    that driver to now point to the secondary (target) device (Backend-B).

    b.  The Service Table in Cinder's database is updated to indicate that a
        replication failover event has occurred, and the driver is currently
        pointing to an alternate target  device.

    In this scenario volumes that were replicated should still be accessible by
    tenants.  The usage may or may not be restricted depending on options
    provided in the failover command.  If no restrictions are set we expect to
    be able to continue using them as we would prior to the failure event.

    Volumes that were attached/in-use are a special case in this scenario and
    will require additional steps.  The Tenant will be required in this case to
    detach the volumes from any instances manually.  Cinder does not have the
    ability to call Nova's volume-detach methods, so this has to be done by the
    Tenant or the Admin.

    c. Freeze option provided as an argument to Failover
       The failover command includes a "freeze" option.  This option indicates
       that a volume may still be read or written to, HOWEVER that we will not
       allow any additional resource create or delete options until an admin
       issues a "thaw" command.  This means that attempts to call
       snapshot-create, xxx-delete, resize, retype etc should return an
       InvalidCommand error.  This is intended to try and keep things in as
       stable of a state as possible, to help in recovering from the
       catastrophic event.  We think of this as the backend resources becoming
       ReadOnly from a management/control plan perspective.  This does not mean
       you can't R/W IO from an instance to the volume.

5.  How to get back to "normal"
    a.  If the original backend device is salvageable, the failover command
    should be used to switch back to the original primary device.  This of
    course means that there should be some mechanism on the backend and
    operations performed by the Admin that ensures the resources still exist on
    the Primary (Backend-A) and that their data is updated based on what may
    have been written while they were hosted on Backend-B.  This indicates that
    for backends to support this something like 2-way replication is going to
    be required.  For backends that can't support this, it's likely that we'll
    need to instead swap the primary and secondary configuration info
    (Reconfigure making Backend-B the Primary).



It's important to emphasize, if the volume is not of type "replicated" it will
NOT be accessible after the failover.  This approach fails over the entire
backend to another device.


Proposed change
===============

One of the goals of this patch is to try and eliminate some of the challenges
with the differences between manage and unmanaged replication targets.  In this
model we make this easier for backends.  Rather than having some volumes on
one backend and some on another and not doing things like stats update, we now
fail over the entire backend including stats updates and everything.

This does mean that non-replicated type volumes will be left behind and
inaccessible (unavailable), that's an expectation in this use case (the
device burst into flames).  We should treat these volumes just like we
currently treat volumes in a scenario where somebody disconnects a backend.
That's essentially what is happening here and it's no different really.

For simplicity in the first iteration, we're specifying the device as a driver
parameter in the config file and we're not trying to just read in a secondary
configured backend device.

    [driver-foo]
    volume_driver=xxxx
    valid_replication_devices='remote_device={'some unique access meta}',...

NOTE That the remote_device access MUST be handled via the
configured driver.

* Add the following API calls
  replication-enable/disable 'backend-name'

  This will issue a command to the backend to update the capabilities being
  reported for replication.

  replication-failover [--freeze] 'backend-name'
  This triggers the failover event, assuming that the current primary
  backend is no longer accessible.

  replication-thaw 'backend-name'
  Thaw a backend that experienced a failover and is frozen

Special considerations
-----------------------

* async vs sync
  This spec does not make any assumptions about what replication method
  the backend uses, nor does it care.

* transport
  Implementation details and the *how* the backend performs replication
  is completely up to the backend.  The requirements are that the interfaces
  and end results are consistent.

* The Volume driver for the replicated backend MUST have the ability to
  communicate with the other backend and route the calls correctly based on
  what's selected as the current primary.  One example of an important detail
  here is the "update stats" call.

  In the case of a failover, it is expected that the secondary/target device is
  now reporting stats/capabilities, NOT the now *dead* backend.

* Tenant visibility
  The visibility by tenants is LIMITED!!!  In other words the tenant
  should know very little about what's going on.  The only information that
  should really be propogated is that the backend and the volume is
  in a "failed-over" state, and if it's "frozen".

In the case of a failover where volumes are no longer available on the new
backend, the driver should raise a NotFound Exception for an API calls that
attempt to access them.


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

One appealing option would be to leave Cinder more cloud-like and not even
offer replication.

Data model impact
-----------------

We'll need a new column in the host table that indicates "failed-over" and
"frozen"  status.

We'll also need a new property for volumes, indicating if they're failed-over
and if they're frozen or not.

Finally, to plan for cases where perhaps a backend has multiple replication
targets, we need to provide them a mechanism to persist some ID info as to
where the fail-over was sent to.  In other words, make sure the driver has
a way to set things back up correctly on an init.

REST API impact
---------------

replication-enable/disable 'backend-name'
  This will issue a command to the backend to update the capabilities being
  reported for replication.

replication-failover [--freeze] 'backend-name'
  This triggers the failover event, assuming that the current primary
  backend is no longer accessible.


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

We'd certainly want to add a notification event that we "failed over"

Also freeze/thaw, as well as enable/disable events.

Other end user impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* Does this change have an impact on python-cinderclient? What does the user
  interface there look like?

TBD

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

* Need Horizon support

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

Obviously this is going to need docs and devref info in cinder docs tree


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

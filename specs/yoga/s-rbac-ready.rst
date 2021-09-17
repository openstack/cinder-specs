..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
The Return of Make Cinder Consistent and Secure RBAC Ready
==========================================================

https://blueprints.launchpad.net/cinder/+spec/s-rbac-ready-y

Revise Cinder's policy definitions to support the
`Consistent and Secure Default RBAC` community goal [1]_.

Problem description
===================

Cinder policies currently rely only on roles and don't recognize scope, which
was introduced into Keystone in Queens.  As a result, implementing something
like an administrator who can only perform nondestructive actions in Cinder
is very complicated and error-prone.
(See, for example, `Policy configuration HowTo
<https://docs.openstack.org/cinder/latest/configuration/block-storage/policy-config-HOWTO.html>`_
in the Cinder documentation.)

Using token scope and other Keystone Queens-era improvements such as role
inheritance, it is possible to define policy rules that recognize a set of
useful "personas".  If all OpenStack services define policy rules to support
this (which is the "consistent" part), operators will not have to rewrite
policies in an attempt to create such personas themselves, but can instead
use tested default policies (which is the "secure" part).

Use Cases
=========

Some examples:

* An operator wants to have an administrator who can perform only
  nondestructive actions.
* An operator wants to have different levels of end user in each project
  (for example, a "project manager" who can do more in the project
  than just a "project member").
* An operator wants to have an system administrator who can manage the
  cinder services, but cannot mess with resources owned by projects that
  the administrator isn't a member of.
* An operator wants to have an customer support administrator who can can
  perform admin-only actions on resources owned by various projects, but
  who cannot modify cinder services.

Proposed change
===============

There's a long comment in the `base policy definition file
<https://opendev.org/openstack/cinder/src/commit/fae0e8dcb430bfe2d00b5360c56aa2e936f5f78c/cinder/policies/base.py>`_
in the cinder code repository that gives a fairly precise description of
how the policy changes made during the Xena cycle would be continued into
Yoga.  Unfortunately, however, as services have begun implementing Consistent
and Secure RBAC, some problems in the initial proposal have been identified,
resulting in a direction change for the effort [2]_.  This means that the
above-mentioned strategy for cinder in the Yoga cycle [3]_ must be completely
revised.

The direction change affects cinder in the following ways:

* The "system-\*" personas are defined to act on the service itself (not
  on project-owned resources) to the greatest extent possible.  (In
  Xena, these personas were envisioned to act as a cinder super-user and
  hence be able to perform all operations, both on project resources and
  on the system itself.)
* The "project-admin" persona is now intended to be an administrator (that
  is, a representative of the operator--definitely not an end user) who can
  perform admin type actions on project-owned resources (for example,
  migrating a volume).
* A new "project-manager" persona is introduced.  This is intended to be an
  end user who has some extra privileges beyond those of a "normal" user
  in a project.  For example, a project-manager could be given the ability
  to set a volume type as the default type for the project.

In Yoga, we'll do the following:

* The Cinder `Policy Personas and Permissions` document merged in Xena [4]_
  contained forward-looking information about what would be done in Yoga.
  Unfortunately, this is no longer accurate.  The stable/xena version of
  this document should be revised to describe *only* the Xena changes so
  that operators aren't misled about the direction the effort is taking.

  There is a patch up addressing this:
  https://review.opendev.org/c/openstack/cinder/+/818696

* The Cinder `Policy Personas and Permissions` document in the Yoga
  development branch will need to be revised:

  * The range of actions allowed to the "system-\*" personas will be
    restricted to operations on cinder services.  These will be
    "system-scoped" actions.
  * The range of actions allowed to the "project-admin" persona will
    allow this person to perform administrative operations on resources
    owned by any project on which this person has the 'admin' role.
    These will be "project-scoped" actions.
  * The "project-member" and "project-reader" actions will likely be
    unchanged from Xena, though they will be explicitly "project-scoped"
    in Yoga.
  * There will be some multiple scoped actions.  Both system- and project-
    personas should be able to list volume types, for example (though
    only a system-admin should be able to create a volume type).

* Once the document has been revised, we'll be able to add a 'scope_types'
  field to each policy rule.

* Additionally, we'll be able to update the check strings for all
  policies.

* Ideally, when a project-admin makes a ``GET /v3/volumes?all_tenants=1``
  call, for example, the response should include the volumes owned by all
  and only the projects on which that project-admin has the 'admin' role.
  For Yoga, we will allow anyone with the 'admin' role to see the volumes
  in *all* projects (as is the case now).  (See the discussion of "Listing
  project resources across the deployment" [5]_ in the community goal
  statement for why it's being done this way for Yoga.)

At this point, we will have completed Phase 1 [6]_ of the community goal.
This will allow cinder to ship with the oslo.policy configuration option
``enforce_scope=True`` in the Z release.  (The importance of this is that
Phase 3 [7]_ of the community goal cannot be implemented until a service
requires ``enforce_scope=True``.)

We will plan to do Phase 2 [8]_ of the community goal during the Z
development cycle.  See the community goal document for details.

As mentioned above, by completing Phase 1 in Yoga, we should also be in
a positon to complete Phase 3 during the Z development cycle.

Alternatives
------------

Do nothing.  Despite all the work we did in Xena, this is still not really an
option, because it's not possible for some projects to use scoping and others
to ignore it while at the same time having consistent personas across projects.
So if we don't upgrade our policy definitions, we will block all OpenStack
clouds from being able to use Consistent and Secure Policies.

Data model impact
-----------------

None.

REST API impact
---------------

The ability to make various Block Storage API calls is already governed by
policies.

Security impact
---------------

Overall this should improve security by presenting operators a suitable
set of "personas" that will function consistently across project that they
can use out of the box instead of implementing their own.

Active/Active HA impact
-----------------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None, we already have to make calls out to keystone to validate tokens
and retrieve user privileges.

Other deployer impact
---------------------

None.

Developer impact
----------------

None, other than doing the implementation work.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* rosmaita
* abishop

Other contributors:

* tosky
* enriquetaso
* eharney
* geguileo
* whoami-rajat
* jobernar

Work Items
----------

* Update the Cinder *Policy Personas and Permissions* document in the
  stable/xena branch.
* Update the Cinder *Policy Personas and Permissions* document in the
  master for the Yoga policy changes described above.
* Add the appropriate ``scope_types`` to all policy rules.
* Update policy checkstrings to separate system policies from project
  policies.
* Update all tests to reflect the above changes, adding new tests as necessary.
* Client changes to support system scope:
  https://review.opendev.org/c/openstack/python-cinderclient/+/776469

Dependencies
============

None, the required changes to complete Phase 1 merged long ago in Keystone
and oslo.policy.

Testing
=======

* Continue to use the testing framework developed in Xena.

* We continue to have the stretch goal (mentioned in the Xena spec) to have
  testing for secure RBAC in the cinder-tempest-plugin, but we do not
  consider it a requirement for successful completion of this spec.

Documentation Impact
====================

The primary user-facing documentation for Cinder is `Policy Personas and
Permissions` in the `Cinder Service Configuration Guide`.

Additionally, we expect that there will be more general documentation for
operators in the Keystone docs given the OpenStack-wide nature of this effort.


References
==========

.. [1] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html

.. [2] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#direction-change

.. [3] https://opendev.org/openstack/cinder/src/commit/fae0e8dcb430bfe2d00b5360c56aa2e936f5f78c/cinder/policies/base.py#L193-L248

.. [4] https://docs.openstack.org/cinder/xena/configuration/block-storage/policy-personas.html

.. [5] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#listing-project-resources-across-the-deployment

.. [6] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#phase-1

.. [7] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#phase-3

.. [8] https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#phase-2

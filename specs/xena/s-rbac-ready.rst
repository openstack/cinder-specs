..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Make Cinder Consistent and Secure RBAC Ready
============================================

https://blueprints.launchpad.net/cinder/+spec/s-rbac-ready-x

Revise Cinder's policy definitions to support the community-wide
Consistent and Secure RBAC effort.

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
  (for example, a "project administrator" who can do more in the project
  than just a "project member").

Proposed change
===============

Make changes to the cinder default policies so that the default configuration
recognizes the keystone 'reader' role and treats it appropriately.  This will
entail rewriting all policies that don't govern "Admin API" calls.

For Xena, we'll implement three personas using project-scope only.  What
this means is that any cinder user must have a role on a project in order
to pass policy checks.  This is basically what we have now, except that
we'll distinguish the 'member' role on a project as distinct from a 'reader'
role on a project.

.. note::
   What exactly "personas" are and what they can do relative to the Block
   Storage API are described in the `Policy Personas and Permissions
   <https://docs.openstack.org/cinder/xena/configuration/block-storage/policy-personas.html>`_
   document.

Additionally, we'll address the following issues:

* New tests will need to be added, because previously we only needed to
  distinguish between a "cinder administrator" and an "end user", whereas now
  we'll have to make distinctions between a larger number of "personas" who can
  make API calls.

* We currently have some unrestricted policies (that is, the check string is
  ``""``).  Their checkstrings will be rewritten to something more specific and
  appropriate.

* We currently have some single policies that govern create, read, update,
  and delete operations on cinder resources.  These will need to be split
  up into finer-grained policies so that a user with only a 'reader' role
  can read a resource without modifying it.

See the "Implementation Strategy" outlined in the `Policy Personas and
Permissions
<https://docs.openstack.org/cinder/xena/configuration/block-storage/policy-personas.html>`_ for more details.


Alternatives
------------

Do nothing.  This is not really an option, however, because in order to keep
backward compatibility with legacy policy configuration, it's not possible
for some projects to use scoping and others to ignore it.  So if we don't
upgrade our policy definitions, we will block all OpenStack clouds from
being able to use Consistent and Secure Policies.

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
  lbragstad
  rosmaita

Other contributors:
  tosky
  enriquetaso
  abishop
  eharney
  geguileo
  whoami-rajat
  jobernar

Work Items
----------

* Natural language description of the (eventual) default configuration. This
  will be helpful to operators, but has also located some deficiencies in
  the current cinder policies. It also gives us something against which to
  validate tests of the default policies.
  https://review.opendev.org/c/openstack/cinder/+/763306

* Policy update patches:
  https://review.opendev.org/q/project:openstack/cinder+topic:secure-rbac

* Testing patches. Groundwork patch is
  https://review.opendev.org/c/openstack/cinder/+/805316

  We'll work with the current structure in cinder of individual files
  in the cinder/policies directory for specific sets of policies, and
  add the tests to the same patch where the policies are redefined.

* Tempest testing. Groundwork patch is
  https://review.opendev.org/c/openstack/cinder-tempest-plugin/+/772915

  Initial test patches:
  https://review.opendev.org/q/project:openstack/cinder-tempest-plugin+topic:secure-rbac


Dependencies
============

None, the required changes in Keystone and oslo.policy merged long ago.

Testing
=======

* We'll need a flexible testing framework because we are going from 2
  personas in Wallaby to 3 personas in Xena to 5 personas in Yoga, and
  we will need to be sure that the deprecated policy checkstrings continue
  to work appropriately until they are removed.  Thus some kind of
  ddt-based approach where we have some base tests and run all the
  different kinds of users through them makes a lot of sense.

* As a stretch goal, because of the complexity of the policy configuration,
  it would be good to have testing in the cinder-tempest-plugin so that each
  persona can be tested against a real Keystone instance.  This would also
  allow using tempest for testing the consistency of the Secure and Consistent
  RBAC across openstack projects.

Documentation Impact
====================

The primary documentation for Cinder will be:
https://review.opendev.org/c/openstack/cinder/+/763306

We expect that there will be more general documentation for operators
in the Keystone docs given the OpenStack-wide nature of this effort.


References
==========

Summary of the general status of the effort at the time of the Xena
PTG, including links to more information:
http://lists.openstack.org/pipermail/openstack-discuss/2021-April/022117.html

What the personas are and how they are intended to work in Cinder are
described in https://review.opendev.org/c/openstack/cinder/+/763306

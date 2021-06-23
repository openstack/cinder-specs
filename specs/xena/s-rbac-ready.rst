..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Make Cinder Consistent and Secure RBAC Ready
============================================

https://blueprints.launchpad.net/cinder/+spec/s-rbac-ready

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

Make changes to cinder so that it can recognize the scope of a token and
add policy checkstrings to implement the "personas" described in
https://review.opendev.org/c/openstack/cinder/+/763306

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

* Policy update patches (adding project scope):
  https://review.opendev.org/q/project:openstack/cinder+topic:secure-rbac

* Testing patches. Groundwork patch is
  https://review.opendev.org/c/openstack/cinder-tempest-plugin/+/772915

  Initial test patches:
  https://review.opendev.org/q/project:openstack/cinder-tempest-plugin+topic:secure-rbac

* Client changes to support system scope:
  https://review.opendev.org/c/openstack/python-cinderclient/+/776469

* Relax the cinder REST API to handle system scope:
  https://review.opendev.org/c/openstack/cinder/+/776468

  (For more about this, see the `Xena PTG discussion
  <https://wiki.openstack.org/wiki/CinderXenaPTGSummary#Consistent_and_Secure_RBAC>`_.)


Dependencies
============

None, the required changes in Keystone and oslo.policy merged long ago.

Testing
=======

* Because of the complexity of the policy configuration, testing will be
  done mostly in the cinder-tempest-plugin so that each persona can be
  tested against a real Keystone instance.

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

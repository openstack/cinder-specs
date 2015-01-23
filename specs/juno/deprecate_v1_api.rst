..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Deprecate Cinder V1 API
==========================================

https://blueprints.launchpad.net/cinder/+spec/deprecate-v1-api

The Cinder V1 API should be deprecated as v2 has been developed back in
Grizzly and stable. Newer features are being developed to v2, that v1 will
never have.

Problem description
===================

The v2 API switch involves some changes from clients to make things more
consistent like `display_name` becoming `name` and `display_description` and
`description`. This is done on both volume and snapshot controllers. It is
assumed there are many clients out there still supporting v1, as well as many
deployed clouds still using v1 that would need to make some changes to ease the
switch.

Use Cases
=========

Proposed change
===============

Leave v1 enabled for Juno, give a warning for enabling it though in Cinder API
service startup. Both /v1 and /v2 can work at the same time which can allow
time to switch clients over in the user's pace. In K, turn off v1.

Alternatives
------------

n/a

Data model impact
-----------------

n/a

REST API impact
---------------

/v1 will continue to work as normal and serve incoming requests.

Security impact
---------------

n/a

Notifications impact
--------------------

n/a

Other end user impact
---------------------

/v1 will continue to work as normal and serve incoming requests. If the end
user hits <ip>:8776/ they will see v2 listed as current and v1 listed as
deprecated.

Performance Impact
------------------

n/a

Other deployer impact
---------------------

The deployer will have to make sure they have enable_v1_api=true in their
cinder.conf. In versions older than Juno, enable_v1_api was default to true,
but Juno will have this option set to false by default.

Developer impact
----------------

n/a

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <thingee>

Work Items
----------

* Have devstack set enable_v1_api=true in lib/cinder.
* Add changes to grenade to set enable_v1_api=true.
* Add v2 support to Nova when using Cinder client, but to also support v1 if
  still enabled.
* Add deprecation warnings to Cinder API for enabling v1 API, and set
  enable_v1_api to default to false.
* Update documentation API ref/spec pages. Update the ops guide where
  appropriate.

Dependencies
============

* Devstack support for Cinder v2: https://review.openstack.org/#/c/22489/
* Nova support for Cinder for v2: https://review.openstack.org/#/c/43986/
* Devstack defaulting to enable_v1_api=true:
  https://review.openstack.org/#/c/102568
* Make sure greneade tests still pass.


Testing
=======

Unit tests for v1 will still exist. Tempest will still do v1 tests in Juno.


Documentation Impact
====================

V1 Cinder documentation will mention it's deprecated where it's appropriate.
Instructions for upgrade and keeping v1 enabled can also be provided. This
includes the reference, spec, and ops docs.

References
==========

n/a


..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Unit test cases for cinder scripts
==========================================

https://blueprints.launchpad.net/cinder/+spec/unit-test-cases-for-cinder-scripts

Currently, there are no unit tests to test
bin/cinder-{all, api, backup, manage, rtstool, scheduler, volume}.
Adding unit tests for these scripts can help prevent issues similar to
https://review.openstack.org/#/c/79791/, as well as increase test coverage.

Problem description
===================

There are no unit tests to test
bin/cinder-{all, api, backup, manage, rtstool, scheduler, volume}.  Adding unit
tests for these scripts can help prevent issues similar to
https://review.openstack.org/#/c/79791/, where a non-existent module was
imported.  Furthermore, it increases the test coverage for each cinder script.

Proposed change
===============

In order to create unit tests for
bin/cinder-{all, api, backup, manage, rtstool, scheduler, volume}, we have to
move them into cinder/cmd, and use pbr to setup the correct console scripts,
which will call the respective main function of each script under cinder/cmd.
It will allow us to import from cinder.cmd and individually test each command.

nova already have their scripts under nova/cmd and uses pbr to setup the
correct console scripts.  It is also the same with glance, where it has
unit tests similar to the one proposed, i.e.
glance/tests/unit/api/test_cmd.py.

Alternatives
------------

The existing setup can be left as-is and no modifications made.  However, this
alternative opens up the possibility of more issues similar to
https://review.openstack.org/#/c/79791/ being introduced into the cinder code.

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

None

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
  thangp

Other contributors:
  eharney

Work Items
----------

* Move bin/cinder-{all, api, backup, manage, rtstool, scheduler, volume} into
  cinder/cmd/cinder_{all, api, backup, manage, rtstool, scheduler, volume}.py.
* Use pbr entry_points to manage the cinder scripts.
* Create positive and negative unit test cases for each cinder command under
  cinder/cmd, i.e.
  cinder_{all, api, backup, manage, rtstool, scheduler, volume}.


Dependencies
============

None


Testing
=======

The goal is to create positive and negative unit tests cases for each cinder
script that is currently under bin/.


Documentation Impact
====================

Packagers should be aware of the following changes to setup.cfg.

cinder uses pbr to handle packaging.  The cinder scripts that is under the
[files] section will be moved to the [entry_points] section of setup.cfg.
More specifically, this proposal adds console_scripts to the [entry_points]
section of setup.cfg as follows::

[entry_points]
console_scripts =
    cinder-all = cinder.cmd.cinder_all:main
    cinder-api = cinder.cmd.api:main
    cinder-backup = cinder.cmd.backup:main
    cinder-manage = cinder.cmd.manage:main
    cinder-rtstool = cinder.cmd.rtstool:main
    cinder-scheduler = cinder.cmd.scheduler:main
    cinder-volume = cinder.cmd.volume:main

This will cause each console script to be installed that executes the main
functions found in cinder.cmd.

References
==========

* Original code proposed by eharney: https://review.openstack.org/#/c/52229/
* Original issue: https://review.openstack.org/#/c/79791/


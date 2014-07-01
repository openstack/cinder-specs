..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Add log translation hints for cinder
====================================

https://blueprints.launchpad.net/cinder/+spec/log-translation-hints

To update cinder log messages to take advantage of oslo's new feature of
supporting translating log messages using different translation domains.

Problem description
===================

Current oslo libraries support translating log messages using different
translation domains and oslo would like to see hints in all of our code
by the end of juno. So cinder should handle the changes out over the release.

Proposed change
===============

Since there are too many files need to change, so divide this bp into 16
patches according to cinder directories.
	├─cinder
	│  ├─api
	│  ├─backup
	│  ├─brick
	│  ├─common
	│  ├─compute
	│  ├─db
	│  ├─image
	│  ├─keymgr
	│  ├─openstack
	│  ├─scheduler
	│  ├─testing
	│  ├─tests
	│  ├─transfer
	│  ├─volume
	│  └─zonemanager

For each directory's files, we change all the log messages as follows.
1. Change "LOG.exception(_(" to "LOG.exception(_LE".
2. Change "LOG.warning(_(" to "LOG.warning(_LW(".
3. Change "LOG.info(_(" to "LOG.info(_LI(".
4. Change "LOG.critical(_(" to "LOG.info(_LC(".

Note that this spec and associated blueprint are not to address the problem of
removing translation of debug msgs.
That work is being addressed by the following spec/blueprint:
https://review.openstack.org/#/c/100338/

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


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

Work Items
----------

For each directory's files, we change all the log messages as follows.
1. Change "LOG.exception(_(" to "LOG.exception(_LE".
2. Change "LOG.warning(_(" to "LOG.warning(_LW(".
3. Change "LOG.info(_(" to "LOG.info(_LI(".
4. Change "LOG.critical(_(" to "LOG.info(_LC(".

We handle these changes in the following order:
	cinder
	cinder/api
	cinder/backup
	cinder/brick
	cinder/common
	cinder/compute
	cinder/db
	cinder/image
	cinder/keymgr
	cinder/openstack
	cinder/scheduler
	cinder/testing
	cinder/tests
	cinder/transfer
	cinder/volume
	cinder/zonemanager

Add a HACKING check rule to ensure that log messages to relative domain.
Using regular expression to check whether log messages with relative _L*
function.
	log_translation_domain_error = re.compile(
		r"(.)*LOG\.error\(\s*\_LE('|\")")
	log_translation_domain_warning = re.compile(
		r"(.)*LOG\.(warning|warn)\(\s*\_LW('|\")")
	log_translation_domain_info = re.compile(
		r"(.)*LOG\.(info)\(\s*\_LI('|\")")
	log_translation_domain_critical = re.compile(
		r"(.)*LOG\.(critical)\(\s*\_LC('|\")")

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

None

References
==========

[1]https://blueprints.launchpad.net/oslo/+spec/log-messages-translation-domain-rollout
[2]https://review.openstack.org/#/c/70455
[3]https://wiki.openstack.org/wiki/LoggingStandards
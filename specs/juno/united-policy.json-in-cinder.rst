..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
United Policy.json In Cinder
============================

https://blueprints.launchpad.net/cinder/+spec/united-policy-in-cinder

Currently, there is two policy.json files in cinder. One for cinder code,
one for unit test code. It's not convenient for the developer and easy to
miss one. This blueprint is aim to united them. Then unit test code will
use the policy.json in the code.

Problem description
===================

Currently, there is two policy.json files in cinder. One for cinder code,
one for unit test code. It's not convenient for the developer and easy to
miss one.

Proposed change
===============

* Delete the policy.json under the test

* Modify the unittest to use the file /etc/cinder/policy.json

Alternatives
------------

None

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
  xiaoding <xiaoding1@huawei.com>

Work Items
----------

* Delete policy.json in the test

* Modify the unittest to use /etc/cinder/policy.json


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

None

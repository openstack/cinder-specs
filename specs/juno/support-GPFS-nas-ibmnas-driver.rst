..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================================
Extending IBMNAS driver to support NAS based GPFS storage deployments
=====================================================================

https://blueprints.launchpad.net/cinder/+spec/add-gpfs-nas-to-ibmnas

Currently, the ibmnas driver works for an nfs export from Storwize V7000
Unified and SONAS products. It does not have the capability to work with
nfs exports provided from a gpfs server.


Problem description
===================

Currently, the ibmnas driver does not have the capability to work with nfs
exports provided from a gpfs server.

* Lacking this capability will limit the end users from using remote gpfs
  NAS servers as a backend in OpenStack environment.

Use Cases
=========

Proposed change
===============

* Add/Reuse functions in ibmnas.py to support all minimum features listed
  (github.com/openstack/cinder/blob/master/doc/source/devref/drivers.rst)
  for NAS based GPFS server backends.


Alternatives
------------

The existing gpfs driver can be extended to support NAS based gpfs storage
deployments. But this implementation requires many other new funtions to be
introduced, which are already existing and can be reused in ibmnas driver.
Apart from this in future, we have planned to support all NFS/GPFS related
IBM products via ibmnas driver. Hence extending ibmnas driver is more
advantageous than extending gpfs driver.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

No specific security issues needs to be considered. Insecure file permissions
(OSSN-0014) is fixed in the driver and is addressed by
https://review.openstack.org/#/c/101919/

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

This requires an additional option to be configured while deploying
OpenStack with IBMNAS products (sonas, v7ku, gpfs-nas).

* New configuration option needs to be filled in cinder.conf
  ibmnas_platform_type = <sonas> | <v7ku> | <gpfs-nas>

* This change needs to be explicitly enabled on IBMNAS driver CI certification

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sasikanth <sasikanth.eda@in.ibm.com>

Other contributors:
  nilesh-bhosale <nilesh.bhosale@in.ibm.com>

Work Items
----------

* Add/Reuse functions in ibmnas.py to support NAS based GPFS storage
  deployments.


Dependencies
============

None


Testing
=======

* Unit tests - Existing test_ibmnas.py will be improved to handle the new
  code changes/functions.
* Tempest tests - No additional testcases needs to be written, this feature
  can be tested with the existing tempest.
* Cinder driver certification tests - Driver certification tests will be
  executed and results will be submitted to the community (as the changes will
  altogether enable a new storage platform).
* CI tests - We are working towards 3rd party CI environment and will
  continuously run tests across the respective hardware platform.


Documentation Impact
====================

ibmnas driver documentation needs to updated with this new configuration
option.

ibmnas_platform_type = <sonas> | <v7ku> | <gpfs-nas>

This option is used for selecting the appropriate backend storage.
Valid values are v7ku for using IBM Storwize V7000 Unified
sonas for using IBM Scale Out NAS and
gpfs-nas for using NAS based GPFS server deployment


References
==========

None

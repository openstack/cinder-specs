..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Attach/detach volumes without Nova
==================================

https://blueprints.launchpad.net/cinder/+spec/use-cinder-without-nova

Services like Ironic want to use Cinder w/o Nova. Also it would be useful
to attach Cinder volumes not only for Nova/Ironic instances: e.g. make
simple CLI to attach volume to some host (VM, workstation, etc) outside
OpenStack cloud.


Problem description
===================

Currently there is not a tool which could automate all the attach/detach
process without using Nova.

Use Cases
=========

* Attach volume to Ironic instance

* Attach volume to any virtual/baremetal host which is not provisioned by Nova
  or Ironic

* Attach volume to Magnum container

* Attach volume to Docker container

Proposed change
===============

Provide a command-line (CLI) tool and Python API which could interact with
Cinder and attach volumes to the local host. It will be implemented as
python-cinderclient extension and won't require to add any dependency if
extension is not used. This extension won't be installed by default. Users
will be able to install it using PIP or OS package management tool::

    pip install python-brick-cinderclient-ext

CLI interface e.g.::

    cinder local-attach <volume_id>
    cinder local-detach [--attachment-id attachment-id] <volume_id>
    cinder get-connector


In case of Ironic when we don't want to install any package into the user's
instances. Users could install package themselves via `pip` or OS package
managers (apt, yum, etc.).


After this command cinder client will call Cinder API to change volume
status to 'in-use' because we don't know when user will attach volume to the
host.

Volume attachment::

    cinder local-attach [--mountpoint /mnt/disk]
                        [--multipath True]
                        [--enforce-multipath True]
                        [--mode rw]
                        <volume_id>

Volume detachment::

    cinder local-detach [--attachment-id id]
                        [--multipath True]
                        [--enforce-multipath True]
                        [--device-info device-info]
                        <volume_id>

'attachment_uuid' is required option only for multi-attachment scenario.

Detach procedure should contain the following steps:

* disconnect_volume

* terminate_connection with a 'force' flag

* detach


Force detach feature is out of scope of this spec and will be described in a
separate spec.


Get connector details:

    cinder get-connector

Since mount/unmount is admin only operation we need to run python-cinderclient
as root to not add oslo.rootwrap dependency.

Alternatives
------------

1. Cinder already has all needed APIs. Any API consumer could use existing
   methods to implement attach/detach actions without Nova.

2. We can implement needed API inside new python-brickclient.

3. We can introduce new binary (e.g.: brick) inside python-cinderclient
   project.
   In such case we'll have two different clients inside single repository.
   From the packager's point of view it will increase numbers of possible
   python-cinderclient packages. E.g.: python-cinderclient-iscsci,
   python-cinderclient-rbd, etc.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

TBD

Notifications impact
--------------------

None

Other end user impact
---------------------

This change will provide new CLI options and Python for python-cinderclient.

Performance Impact
------------------

None

Other deployer impact
---------------------

* Deployers could install new package to have new functionality


Developer impact
----------------

* All drivers should implement initialize_connection which could be invoked
  several times without any side-effect. We've filed bugs for some of the
  drivers.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Ivan Kolodyazhny <e0ne>


Work Items
----------

* Implement proof-of-concept based on a python-cinderclient

* Get feedback and complete the spec

* Fix typos in the spec

* Make changes to Cinder API if needed

* Unit-tests and Tempest tests are required

* Functional tests on python-cinderclient gates will be implemented

* New repository for python-brick-cinderclient-ext is needed


Dependencies
============

* os-brick
* python-cinderclient
* Linux: open-iscsi, ceph-common, nfs-common or other tool to attach and detach
  volumes via drivers' protocols
  Windows: tools for Windows to be able to attach and detach volume via
  drivers' protocols.



Testing
=======

* Unit-tests should be implemented

* New python-cinderclient API will be tested via functional tests on gates
  to test attach/detach features without Nova instances.

* Functional tests for Ironic will be implemented to test attach/detach
  features to Ironic instances.


Documentation Impact
====================

User manual will be updated to contain information about python-cinderclient
extension.


References
==========

* https://review.openstack.org/254878 and related patches

* http://lists.openstack.org/pipermail/openstack-dev/2015-July/068763.html

* https://etherpad.openstack.org/p/liberty-cinder-ironic-integration

* https://blueprints.launchpad.net/ironic/+spec/volume-connection-information

* python-brickclient Proof-of-Concept implementation: https://github.com/e0ne/python-brickclient

* Cinder Meeting Minutes: http://goo.gl/ztuYTa

* http://eavesdrop.openstack.org/meetings/cinder/2015/cinder.2015-11-11-16.00.log.html

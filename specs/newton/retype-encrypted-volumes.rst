..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Retype volumes with different encryptions
=============================================

https://blueprints.launchpad.net/cinder/+spec/retype-encrypted-volume

Enable the function that changing volume type of a volume [1] to another
type with different encryptions.

Problem description
===================

Currently Cinder prevents retyping volumes to a volume type with different
encryptions.

Use Cases
=========

* Customers use unencrypted volumes, but later they would like
  to change the volumes to encrypted.
* Customers use encrypted volumes and later want to change to unencrypted.
* Customers want to change encryptions of a volume.

Proposed change
===============

Allow retype between encrypted and unencrypted volumes.
Same as current retype mechanism, it allows to retype volumes in available
and in-use volumes.

If a volume is in available status, the detailed process will be:

* Create a new volume according to new volume_type.
* Map the two volumes to the volume host.
* Open the device with dm-crypt if volumes are encrypted. This is done
  through os-brick/encryptors [2].
* Copy data from original volume to new volume.
* Close dm-crypt and detach the volumes.
* Delete original volume in backend storage.

If a volume is in-use status, nothing needs to change except the bug fix [3].

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

With the feature, it allows users to retype a volume to different
encryptions.

Security impact
---------------

Cinder needs to access encryption keys and decrypt the data.

Notifications impact
--------------------

A flag will be added to current retype notification to show whether
it needs encryption change.

Other end user impact
---------------------
During retyping volumes with different encryptions, Cinder needs to get key.
But Barbican can be configued only to give key materials to tenants, not admin.
This may lead that admin can't retype volumes successfully. In such cases,
Cinder will catch the exception, log the error. The volume to retype will be
set to original state.
As os-brick/encryptors doesn't work on RBD, Sheepdog volumes, the function to
retype such volumes to different encryptions will fail, and volumes will be
set to original state.

Performance Impact
------------------

It adds the step of encrypting/decrypting data during retype process,
and the impact is dependent on the performance of encryption.

Other deployer impact
---------------------

The feature is dependent on Castellan [4]. Meanwhile, Barbican [5] is currently
the only key manager backend supported by Castellan. Both the two packages
are needed.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  LisaLi <xiaoyan.li@intel.com>


Work Items
----------

* Remove current limitation which disallows the retype.
* Attach/detach encrypted volume through dm-crypt.


Dependencies
============

None


Testing
=======

Unit tests need to be created to cover the code change that
mentioned in "Proposed change".
New tempest test cases will be added after current retype test [6].


Documentation Impact
====================

The cinder API documentation will need to be updated to describe the change.

References
==========

* [1]: http://docs.openstack.org/cli-reference/cinder.html#cinder-retype
* [2]: https://review.openstack.org/#/c/247372/
* [3]: https://review.openstack.org/#/c/252809/
* [4]: https://github.com/openstack/castellan
* [5]: https://github.com/openstack/barbican
* [6]: https://review.openstack.org/#/c/195443/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


====================================
Delete multiple volume metadata keys
====================================

https://blueprints.launchpad.net/cinder/+spec/delete-multiple-metadata-keys

Problem description
====================
The current implementation of Cinder API functionality supports deleting of
multiple volume metadata as multiple requests for deleting only one metadata
key with a request. The more keys are necessary to remove, the more API
requests and DB queries are needed. It would be more efficient to delete
multiple metadata keys with a single request at a time.

Use Cases
=========

Deleting multiple volume metadata keys with a single request.

Proposed change
================
To delete multiple metadata items without affecting the remaining ones,
just update the metadata items with the updated complete list of ones
(without items to delete) in the body of the request. On success, the
server responds with a 200 status code.

.. note:: A PUT request should use etags to avoid the lost update problem.

Alternatives
------------
Have a request that deletes all keys rather than a specified list as an
option.

Data model impact
-----------------
None

REST API impact
---------------
The existing API for update metadata will be used in this case. The API
call will fail in the case some of the items that are specified to delete
does not exist. The API call will send the set of keys that are supposed
to be updated and ignore missing keys.

Security impact
---------------
None

Notifications impact
--------------------
One delete notification will be emitted per metadata item deleted as today.

Other end user impact
---------------------
A user will have new API.

Performance Impact
------------------
The deleting of a volume metadata keys with a single request allows to
improve performance by reducing DB calls. It's very important in case with
many keys. Better performance due to fewer request round trips.

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
Yuriy Nesenenko(ynesenenko@mirantis.com)

Work Items
----------
* Extend python-cinderclient to support new API.
* Add unit and tempest tests.

Dependencies
============

Depends on Cinder API microversion.
Depends on API WG patch https://review.openstack.org/281511/
Depends on API WG patch https://review.openstack.org/301846/

Testing
=======

Unit and functional tests are needed to ensure response is working correctly.

Documentation Impact
====================

Documents concerning the API will need to reflect these changes.

References
==========

None

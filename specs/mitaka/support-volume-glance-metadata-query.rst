..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Support query volume detail with glance metadata
================================================

https://blueprints.launchpad.net/cinder/+spec/support-volume-glance-metadata-query

Provide a function to support query volume detail filter by glance metadata.

Problem description
===================

The purpose of this feature is to make user to query volume detail more
conveniently. User can query a specific bootable volume quickly filtering by
image_name or other glance metadata.

Use Cases
=========

At large scale deployment, there could be many bootable volumes in a tenant.
So if user want query some bootable volume detail filtering by the image name
or other info came from glance metadata, they could use this feature to query
it more conveniently.
No need to list all volumes and find what you want tiring.

Proposed change
===============

* Add DB query filter using volume_glance_metadata in api of sqlaclchemy.

* User can use glance metadata to filter volume detail in cinder api.
  The query url is like this::

      "volumes/detail?glance_metadata={"image_name":"xxx"}"

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

Add query filter support using glance metadata:
  * GET /v2/{project_id}/volumes/detail?glance_metadata={"image_name":"xxx"}

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

None

Performance Impact
------------------

Search a lot of glance metadata may take longer than other querying.
It may need add new index to column key and value in volume_glance_metadata
table to improve searching performance.

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
  wanghao<wanghao749@huawei.com>


Work Items
----------

* Implement code in db query filter.
* Update cinderclient to support this function.
* Add change API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change".


Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Support get volume metadata summary
===================================

https://blueprints.launchpad.net/cinder/+spec/metadata-for-volume-summary


Support get volumes' metadata through volume summary API.

Problem description
===================

Currently, Cinder supports filter volumes with metadata. But in some case,
users don't know what metadata all the volumes contain or what metadata is
valid to filter volumes. Then users need to show the volumes one by one to get
the correct metadata, this is really a heavy and unfriendly way. Imaging that
if there are hundreds of volumes, admin users will take a very long time to
query all metadata.

Use Cases
=========

1. For users, they can get all the metadata easily just through one API
request.
2. For dashboard, such as Horizon, it can use this metadata information to show
end users a dropdown list.

Proposed change
===============

1. DB layer change:
    All the volumes' metadata can be got by the sql query.

2. API layer change:
    Add a new micro version. Add "metadata" to volume-summary API response
    body. The body will be like::

    "metadata": {"key1": ["value1"],"key2": ["value2", "value3"]}

Alternatives
------------

Leave as it is. Let the operators get volumes metadata by themselves through
some peripheral ways. Such as, create a script to call volume-list-detail API
and then analyse the result one by one.


Data model impact
-----------------

None

REST API impact
---------------

A new microversion will be created.

Cinder-client impact
--------------------

Now both OpenStackClient and CinderClient don't support volume-summary
command. We can add them as well.

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

There is a little performance influence about volume-summary API since a new
sql query action will be added.

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
  wangxiyuan(wangxiyuan@huawei.com)

Work Items
----------

* Add a new microversion.
* Add volumes' metadata to the volume-summary API's response body.
* Add client side support.

Dependencies
============

None

Testing
=======

* Add unit tests.

Documentation Impact
====================

Update API documentation.

References
==========

None

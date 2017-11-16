..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Use 'LIKE' operator to filter resource
======================================

https://blueprints.launchpad.net/cinder/+spec/support-regexp-based-query


This blueprint proposes changing some filter behaviour to use 'LIKE'
expression on some columns(name, description, etc).

Problem description
===================

Currently, we could only filter cinder resource by exact match, and this is
not flexible enough when user would like to retrieve some resource whose name
or attribute is partly alike, especially for the users who have lots of
volumes. As it's already introduced in many other projects, we can take
advantage of those some existing mechanism, nova `[1]`_ and ironic `[2]`_.

Use Cases
=========

Possible use case is to support client bulk operation, collect the
desired resources with single command.

Proposed change
===============

Although we can provide filtering resource based on regex filter, but there
is a possibility that we could have ReDos `[3]`_ attack. Considering the
'LIKE' operator is flexible and safe enough for this case. We could only
introduce 'LIKE' operator ('NOT LIKE' is another useful operator, but it's
not as common as 'LIKE'). And we can easily apply this filter at the
existing common filtering function individually with a decorator::

    @apply_like_filters(model=models.Volume)
    def _process_volume_filters(query, filters):
        pass

This spec intends to support 'LIKE' based filter on some specified resources
and columns, as we would have the generalized resource filtering feature
`[4]`_, this one will take advantage of that spec in order to make it
configurable for administrators, so this is what we would finally have in our
filters' configuration file ::

    {
        "volume_filters": ["availability_zone", "status", "name~"]
    }

'~' here means the attribute 'name' supports query with like operator (not
indicating the regex operator),  once this is configured, the input
key-value pair ``name~=value`` from API will finally be translated into this
sql statement::

    # '%' is automatically appended at both sides
    SELECT * FROM volumes
    WHERE name LIKE '%value%'

so with that switch on, end user could filter volume with these inputs below::

    cinder list --filters name=volume_preserved   #exact matches
    cinder list --filters name~=volume_preserved  #inexact matches

assuming we have two volumes in the name of 'volume_preserved_01',
'volume_preserved_02', so with first command we will get none of
them while last command would have both of them.


Alternatives
------------

There is an option that we can deploy searchlight which mainly focus on
various cloud resource querying, and that's a more widely topic. But we
should also consider the cloud environment that doesn't have a
searchlight.

Also, there is another option that user can gather all the raw data and
do the filtering on their own, but it's obviously that that is what we try
to avoid, cause it costs a lot of unnecessary resource expense.

Data model impact
-----------------

None

REST API impact
---------------

Microversion bump is required for this change.

Cinder-client impact
--------------------

Help text will be updated to advertise this change.

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
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Update the generalized resource filtering
  logic to accept new symbol '~' in config file.
* Update target object's common filter method.
* Add related unit testcases
* Update cinder-client and OSC.

Dependencies
============

Depended on generalized resource filtering `[4]`_

Testing
=======

* Add unit tests to cover filter process change.

Documentation Impact
====================

Update API documentation.

References
==========

_`[1]`: https://review.openstack.org/#/c/45026/
_`[2]`: https://review.openstack.org/#/c/266688/
_`[3]`: https://en.wikipedia.org/wiki/ReDoS
_`[4]`: https://review.openstack.org/#/c/441516/


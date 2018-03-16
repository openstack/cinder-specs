..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Support filter backend based on operation type
==============================================

https://blueprints.launchpad.net/cinder/+spec/support-mark-pool-sold-out

This blueprint proposes to support filter backend based on operation type.

Problem description
===================

Now our filter and weigher plugins' algorithm are all designed based on
two factors, backend information and user input. But there could be
another factor that may also affect the scheduler result, operation
type. For instance, if administrators want to disable some of the
backend pool only for creating new volume operation, it's impossible
based on our current logic.

Use Cases
=========

We have discussed the concept of **sold out** during Rocky PTG `[1]`_ and
that word is more related to administrative operation, when most of an
specific pool's capacity has been consumed, administrators need the
functionality to mark the backend pool sold out, that is to say, it
will become unavailable for allocating new volumes, but end users still
can perform any operations on their existing resources (extend volume,
create snapshot).

Adding this feature in cinder will help cloud vendors write their
own plugins which can filter backend based on the operation type as well
as backend information.

Proposed change
===============

New attribute ``operation`` will be added into ``RequestSpec`` object
and plugins can filter backend based on backend states as well as
request type:

.. code-block:: python

        def backend_passes(self, backend_state, filter_properties):
            #get request type from filter_properties
            spec = filter_properties.get('request_spec', {})
            request_type = spec.get('operation', None)
            if request_type in ['create_volume']:
                #check backend state when request is for creating new volume
                return backend_state.free_capacity_gb > MIN_REQUIRED_GB
            if request_type in ['extend_volume', 'retype_volume']
                #perform other validation when request is ......

**NOTE**: This spec doesn't propose to update any plugin's logic, the code
above only intends to demonstrate how the ``operation`` can be used.

Based on our current features, the value of ``operation`` can be one of
these below:

1. create_group
2. manage_existing
3. extend_volume
4. create_volume
5. create_snapshot
6. migrate_volume
7. retype_volume
8. manage_existing_snapshot

And we will inject scheduler manager's corresponding methods in order to
append this info to ``RequestSpec`` automatically:

.. code-block:: python

        @append_operation
        def extend_volume(self, context, volume, new_size, reservations,
                          request_spec=None, filter_properties=None):
            #other existing codes

Alternatives
------------

The alternative is to implement a custom scheduler manager and configure it
in ``scheduler_manager`` and then use either a custom filter or the JSON
filter.

Data model impact
-----------------

``RequestSpec`` will be updated to store this new attribute.

REST API impact
---------------

None

Cinder-client impact
--------------------

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

Developers can develop new scheduler plugins based
on backend state as well as request type.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Add ``operation`` to those actions which will be scheduled
  to scheduler to perform filtering and weighing logic.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

* Update the developer document to mention the new attribute
  will be delivered to scheduler plugins as well.

References
==========


_`[1]`: https://etherpad.openstack.org/p/cinder-ptg-rocky-wednesday
_`[2]`: https://review.openstack.org/#/c/308869/

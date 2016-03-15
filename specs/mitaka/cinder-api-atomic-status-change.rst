..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Cinder Volume Active/Active support - API Races
=============================================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion.

One of the reasons for this is non-atomic state transitions in c-api that may
cause race conditions.

This spec proposes a change to do atomic DB changes in Cinder API using
compare-and-swap to prevent them.


Problem description
===================

Current Cinder API Service code has sections where contents from the DB are
retrieved at one point and then checked for validity and in case all checks
pass and the operation can proceed DB is changed accordingly.

Elapsed time between the retrieval and the DB changes depends on the nature of
the validation, in some cases it's just a simple check of the 'status' field,
while in others it requires checking other tables for example to confirm that a
volume is not attached or has no snapshots.

This way of handling checks and changes in the API creates a window of
opportunity for code races to occur. These races would happen when changes are
made to the DB after we have retrieved its contents, leading us to make a
decision based on outdated data and not only change the DB using obsolete data
as reference, but in most cases also incorrectly calling Volume Manager's
Service to perform an operation.

One example of these API races is the extend method in cinder/volume/api.py
where we check at one point the status:

 if volume['status'] != 'available':

Then later on we change the status:

 self.update(context, volume, {'status': 'extending'})

And we finally make an RPC request to perform the operation:

 self.volume_rpcapi.extend_volume(context, volume, new_size, reservations)


Use Cases
=========

Operator would want to avoid unexpected behavior in their clouds, moreover
since these race conditions could lead to data corruption in the storage
back-end.  And these are probably situations more likely to happen in an
Active/Active configuration that can handle a higher volume workload.


Proposed change
===============

Proposed solution is to use compare-and-swap updates, with retries on
Deadlocks, to ensure that we only update the DB when all required conditions
are met.

Basically a compare-and-swap is just a complex DB query that has a condition
added to the update query to ensure that modifications to the DB only occurs if
those conditions are met.  They can be simple conditions like status ==
'available' or more complex conditions referring to other tables.

This solution has been chosen over other alternatives because `performance
numbers`_ in tests show that even under the most extreme conditions in a
multi-master cluster DB configuration they will have great performance results.

The idea is to keep Cinder API service behaving as close as possible to the
original code, so we will fail and succeed under the same conditions.  Even
though reported errors will be slightly different, the only relevant difference
will be when we previously would have had a race condition, and possibly data
corruption, that both API caller would have received an affirmative response to
their request and now one of them will receive a failure as if both operations
had been performed sequentially.

The small difference in how we report errors comes from the fact that on
failure we no longer know which of the conditions triggered the failure.  So in
name of efficiency and code clarity we will be returning a generic error
stating that some of the required conditions were not met and therefore
requested operation could not be completed.

Alternatives
------------

An alternative to returning a generic error on failure, that was rejected for
its complexity, would be to return the exact same errors we are returning now.
For that, we would get updated data from the DB and give it to the validation
method that will check which of the conditions is not met and raise the right
exception with the right error message.  For example this method could be
checking the ``status`` of a volume and raise a message if it is not
``available``, and then check that the volume has not snapshots and if it does
then raise a different exception or the same exception with a different
message.

Skeleton of the procedure::

 result = resource.conditional_update(values, conditions)
 if not result:
     resource = db.get_resource(resource.id)
     raise_right_validation_error(resource)
 rpc_call(resource)

But there is a potential race here, we could be requesting a conditional update
that fails (because the conditions in the DB are not met), and when we retrieve
the data from the DB it has just changed (now it would meet the conditions for
the update), so when we call the method in charge of raising the right
validation error with this changed data it will not find any reason why we
cannot perform the operation, so it will not raise any exception and will just
return control to the caller.

In that case, since we didn't raise an error, we would continue with the rpc
call without having changed the DB, which is a problem.  That is why we need a
loop in the conditional update, to make sure we either succeed on the update or
we raise the error regardless of racing conditions on the data.

Updated skeleton of the procedure::

 while not resource.conditional_update(values, conditions):
     resource = db.get_resource(resource.id)
     raise_right_validation_error(resource)
 rpc_call(resource)

In this implementation it would be a good idea to have a maximum number of
retries instead of just looping until we either update the DB or can properly
raise an error.  That way we would not have an infinite loop if we have an
error in our conditional update or our validation code.

There are also multiple alternatives to compare-and-swap that have been
explored and discarded for being less efficient - as in being slower or
requiring more queries to the DB.

Discarded alternatives are:

- SELECT ...  FOR UPDATE, with retries on Deadlocks, which even though it works
  with multi-master configurations it has more Deadlock retries than proposed
  solution.

- Using a Distributed Locking Manager with different backends to enforce
  exclusive access to resources.

More information on the tests can be found in `performance numbers`_.

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

Performance impact will be negligible, and may even have better performance
since there will be only one query to the DB instead of multiple queries.  For
example on volume deletion, where we now have one query to see the status and
another to get the count of snapshots for that volume.  With the new
compare-and-swap we will only have 1 query that will update the DB if the
status is correct and no snapshot exist.

Other deployer impact
---------------------

None


Developer impact
----------------

Once changes are made to Cinder API methods, all new API methods that are added
will need to conform to this new compare-and-swap way of updating the DB to
prevent new race conditions from entering the code base.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Other contributors:
  Anyone is welcome to help

Work Items
----------

* Atomic update method for DB.

This method will require certain capabilities:

- Basic comparing:
  status = 'available'
  status != 'available'

- Comparing of multiple values:
  status in ['available', 'error']
  status not in ['attaching', 'detaching']

- Handling None values like Python does: When doing a negative comparison
  against a non None value it should return None values as well. So checking
  for 'migration_status' != 'migrating' would also return values that have
  'migration_status' set to None.

- Using additional complex filters: In some cases, like checking for
  attachments, more complex queries may be needed.

- Set update values depending on other fields.  For example when detaching a
  volume you will have to set the status to 'available' if there are no more
  volume attachments for that volume, and set it to 'in-use' if there are more
  attachments (it was multi attached).

- Set update values based on fields on the DB:
  previous_status = status, status = 'retyping'
  size = size + 10


* Atomic update method in Versioned Objects.

Besides exposing the functionality provided by above DB conditional update item
it also needs to:

- Automatically add the ID of the resource to the conditional update.

- If no conditions are given it must assume we want the DB record to be
  unaltered.

- It must allow saving dirty attributes together with the update.

- It must be able to update the versioned object with data that has been
  written to the DB, even on conditional values.


* Update Cinder API Service methods to use compare-and-swap.


Dependencies
============

No openstack dependencies, but it has a SQLAlchemy depencency on version 1.0.10
that includes Parameter-Ordered Updates (`issue #3541`_).

On some DBs' update method is order dependent, so they behave differently
depending on the order of the values, example on a volume with 'available'
status:

   UPDATE volumes SET previous_status=status, status='retyping' WHERE
   id='44f284f9-877d-4fce-9eb4-67a052410054';

Will result in a volume with 'retyping' status and 'available' previous_status
both on SQLite and MariaDB, but

   UPDATE volumes SET status='retyping', previous_status=status WHERE
   id='44f284f9-877d-4fce-9eb4-67a052410054';

Will yield the same result in SQLite but will result in a volume with status
and previous_status set to 'retyping' in MariaDB, which is not what we want, so
order must be taken into consideration.


Testing
=======

Unit tests will be added for Atomic Update methods.


Documentation Impact
====================

None


References
==========

.. _performance numbers: http://gorka.eguileor.com/cinders-api-races/
.. _issue #3541: https://bitbucket.org/zzzeek/sqlalchemy/issues/3541/

* https://etherpad.openstack.org/p/cinder-active-active-vol-service-issues
* https://review.openstack.org/#/c/205834
* https://review.openstack.org/#/c/216377
* https://review.openstack.org/#/c/205835
* https://review.openstack.org/#/c/216378
* https://review.openstack.org/#/c/221441
* https://review.openstack.org/#/c/221442

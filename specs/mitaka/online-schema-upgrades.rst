..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Online DB schema upgrades
=========================

https://blueprints.launchpad.net/cinder/+spec/online-schema-upgrades

One of the priorities for Mitaka cycle is making Cinder able to perform rolling
upgrades. This effort requires a lot of discipline to block any changes that
can break interoperability of different services versions.

Some database migrations can break this interoperability. This spec aims to
provide a guideline of how we handle such changes to make sure Cinder is able
to do rolling upgrades.

Problem description
===================

Non backward compatible database migrations include:

* Non-additive migrations:

  * Column, tables removals
  * Column, tables renames
  * Adding constraints

* Long-running data migrations (for example from one column to another)

All of these should be run when all the services are down. This may not be a
problem for small amount of data, but when we're talking about thousands of
rows, the downtime may be significant.

Use Cases
=========

Deployer would like to be able to perform Cinder upgrade without downtime of
control plane services.

User would like not to experience downtime of control plane when cloud he's
using is getting upgraded.

Proposed change
===============

In general we want to build on Nova's experiences and adapt their solution for
our needs. Detailed description of how Nova is doing DB migrations in a live
way is presented in `Dan Smith's blogpost`_.

At this moment please note that we're not basing on older Nova's approach
called `Online Schema Changes`_ that was automatically doing additive and
contracting DB migrations based on current and desired DB model described in
models.py. This approach would eventually lead to developers not needing to
write migrations manually. We will not adopt this solution as it was considered
experimental, was known for destroying data, wasn't working in late Liberty and
was `recently removed from Nova`_.

The main difference between Cinder and Nova that affects online DB schema
upgrades solution is the fact that in Cinder all the services are querying the
DB. In Nova only nova-conductor has access to the DB. Therefore we need to
modify the approach slightly. The changes force us to span the migrations over
3 releases instead of 2 like in Nova.

First of all we should introduce a unit test that will block migrations that
are contracting. Basically such test is failing when a migration using DROP or
ALTER command is in tree. We will do it in a similar way `like Nova`_ did.

After adding such test we should allow only migrations that are expanding or
additive, that is - are adding new columns or tables. Good part is that since
Juno we've only merged migrations that are like that. This means that we will
need the special way rarely.

And (a little complicated) special way goes as follows. Let's use a real-world
example to make this a little more useful. We'll try to model live DB schema
change that will change the relationship between consistency groups and volume
types to be modelled correctly. This is a n-n relationship that currently is
represented by `volume_type_id` column in `consistencygroups` table that is
storing comma-separated ids of volume types related to the consistency group.
Correct way to represent that is to use `ConsistencyGroupVolumeTypeMapping`
table which maintains connections between CG and VT.

We will be working through 3 releases (+ the current one), so let's identify
them using letters:

* L - current
* M - in development
* N
* O

M release
---------

In L release we have only the old representation of this relationship. In M we
should merge a change that adds `ConsistencyGroupVolumeTypeMapping` model and
required columns in `ConsistencyGroup` and `VolumeType` models that will form
relationships to the table mentioned earlier. This additive migration can be
easily executed on database while L services are running.

As on upgrade we will have L and M releases operating in the same moment we
should modify M's CG versioned object to:

* Write data in both places.
* Read data from old place.

That way we maintain compatibility with M->L (M is writing to old place) and
L->M (M is reading from old place).

Once upgrade is completed admin should use a tool that will finish data
migrations. This tool will be developed in cinder-manage and will operate on
chunks of CG rows (to limit performance impact). It will simply iterate and set
the relationship in new place.

In the end everything is running M services, so everythings writes to both
places and reads from the old one. Also we should have all the data migrated at
this point so for all CGs have relationship defined both in new and old way.

N release
---------

Now on upgrade we will have M and N release versions. N's versioned objects:

* Write data in both places.
* Read data from new place.

This way we maintain compatibility M->N (N is writing in place M is reading)
and N->M (M is writing in place N is reading).

As N is reading from new place, before starting any N's service we need to make
sure that all the CGs are migrated. To enforce that, as a first N's migration
we should merge a dummy one that will block subsequent ones if there are items
that weren't migrated yet.

Please note that we still have old representation in SQLAlchemy model, so we
would be unable to drop the column even if we managed to switch N services to
write only to the new place.

O release
---------

This time we'll have N and O services cooperating.

In O we finally modify the SQLAlchemy model to remove the old
`ConsistencyGroup` `volume_type_id` field. This time we're able to, because
it's guaranteed that all services write and read from new place.  We also
remove it from O's versioned object - it will now read and write data only to
the new place.

After the upgrade we have everything writing and reading only from the new
place. Moreover - SQLAlchemy models in O's services doesn't have a notion of
`volume_type_id` column. Therefore we can have a post-upgrade migration script
that finally drops the column (or we can simply add it as migration in P
release if we want to limit number of steps needed to do upgrades.

Alternatives
------------

We may simply block both contracting and data migrations. This however will
prevent us from fixing bad decisions made in the past, which may hurt the
project, as I don't believe it is mature enough yet.

It's possible to implement `Online Schema Changes`_ to make DB migrations to
have two steps divided automatically. Nova was considering this as
experimental. Moreover the code got removed in Mitaka-1. It also doesn't solve
long-running data migrations problem so doesn't remove all the complications
this spec is proposing.

We can also leave things as they are now and assume you need this downtime to
upgrade. This probably isn't what operators want and all the efforts we've
already did (versioned objects, RPC compatibility) would be wasted.

Data model impact
-----------------

We don't expect impact to the data model itself, but to the way we're doing
migrations of the data model.

Changes in the model would need to be reviewed carefully to make sure they
follow this guideline.

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

We can expect a little performance impact on DB calls because we'll be queuing
and saving additional columns instead of dropping unused ones right away. This
shouldn't be very significant however.

There will be overhead on the DB when executing small chunks of DB migrations
while the cloud is running. This is however a tradeoff, as alternative to that
is downtime of the whole cloud.

Nova does this in a very similar way and they doesn't report performance
issues.

Other deployer impact
---------------------

It will be needed for deployer to be aware of how to do DB upgrade in a correct
way. This will be documented, but knowledge will need to be propagated.

Developer impact
----------------

There will be huge impact on developers and reviewers as both groups will need
to be aware how Cinder is doing DB migrations and write/review the code
accordingly to prevent rolling upgrades feature from breaking.

Implementation
==============

Assignee(s)
-----------


Primary assignee:
  michal-dulko-f (dulek)

Other contributors:
  Engagement from whole core team is needed to make sure the code is reviewed
  also from the perspective of live upgrades.

Work Items
----------

* Add unit test blocking contracting migrations (needed in Mitaka).
* Add online DB data migrations bits to cinder-manage.
* Write developer guide on how to write DB migrations properly.
* Add DB migrations bits to Cinder's upgrade documentation.

Dependencies
============

None for itself. Whole rolling upgrades story depends on RPC compatibility
(API versioning and versioned objects with compatibility mode).

Testing
=======

Partial Grenade tests should be added to make sure upgrading services
one-by-one doesn't break rolling upgrades. This should also cover DB migrations
step.

Documentation Impact
====================

We need to write detailed developer guide on how to write DB migrations in new
model to make sure that we have a clear resource which we can point developers
to.

To educate administrators and deployers on how to do rolling upgrades DB
migrations parts should be added to general upgrade instruction for Cinder.

References
==========

* `Discussion at the Design Summit`_

.. _Online Schema Changes: https://blueprints.launchpad.net/nova/+spec/online-schema-changes
.. _Dan Smith's blogpost: http://www.danplanet.com/blog/2015/10/07/upgrades-in-nova-database-migrations/
.. _recently removed from Nova: https://review.openstack.org/#/c/239922/
.. _like Nova: https://review.openstack.org/#/c/197349/
.. _Discussion at the Design Summit: https://www.youtube.com/watch?v=2BLJMPsaWZg

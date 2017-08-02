..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Provisioning Improvements
=========================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/provisioning-improvements

Cinder provisioning is still a source of pain for everyone: end users, admins,
and developers.  Multiple factors have contributed to our current situation,
like the information being dispersed in different specs, developer's reference,
and even the code, but we also have cases of misinterpreted documentation,
documentation not keeping up with the project's evolution, and even misleading
or incorrect documentation.

This spec will build on those that preceded it on the same topic [1]_ [2]_ and
others related to the subject [3]_ to bring a consolidated and updated view on
the matter as well as add a some minor improvements and fix some issues in
hopes that we can provide a better experience for all involved.


Problem description
===================

Our current situation is quite chaotic, we have volume creations that fail
based on an incorrect capacity calculation that would have succeeded if we had
just received an update on the stats of the backend, we have drivers that are
reporting incorrect data on the stats, and we have volumes that cannot be
created when they should be allowed to.

Before going any further we first need to define the terms we'll be using to
ensure they hold the same meaning for all of us, as this is the source of some
of our current issues.

Disagreement on the mapping of these terms and their description is
understandable, but for the sake of understanding each other we'll hold below
descriptions as true since they were defined as such in our specs and most of
our code.

Improvements on the word used for the terms and field names can be discussed at
another time and updated in the specs, documentation, and code accordingly.

For the sake of completeness and to remove any misunderstandings that could
lead to different implementations on the drivers, which we currently have, the
descriptions will also include some clarifications and examples that may
reference current Cinder code.

Terminology
-----------

GB:
  Even though this is formally known as the symbol representation of a gigabyte
  -decimal unit of measurement- we will be using it throughout the spec and our
  code as the symbol for the gibibyte -binary unit of measurement as defined by
  the International Electrotechnical Commission (IEC), with symbol GiB-, so
  when we talk about 1GB we are talking about 1024MB, and the same applies to
  TB and MB.

Total capacity:
  It is the total physical capacity that would be available in the storage
  array's pool being used by Cinder if no volumes were present.

  This is currently being reported by the drivers as `total_capacity_gb` and,
  as the name indicates, should be reported in GB and with a precision no
  greater than 2 decimals.

  If the storage array has 5TB of space but the pool used in Cinder is limited
  to 1TB then the driver should be reporting a `total_capacity_gb` of 1024GB.

Volume size:
  It is the maximum physical size that a volume can take in the storage array.

  This is referenced throughout the code as `volume_size`.

  For a thick volume the `volume_size` will be the same as the free capacity we
  lost when it was provisioned, whereas for a thin volume it will be greater
  than the space used for the volume in the storage array until the volume gets
  completely full.

Free capacity:
  It is the current physical capacity available in the storage array's pool
  being used by Cinder.  The number and volume sizes of the thin and thick
  volumes that have been provisioned by Cinder or directly in the storage array
  are irrelevant here.

  This is currently being reported by the drivers as `free_capacity_gb` and, as
  the name indicates, should be reported in GB and with a precision no greater
  than 2 decimals.

  If the storage array has 5TB of space with a total of 3TB available for all
  its pools but Cinder is using a pool that has a limit of 1TB of which it has
  already used 400GB and someone has manually created volumes outside of Cinder
  that are currently using 124GB of space, then the driver should be reporting
  a `free_capacity_gb` of 500GB (1TB = 1024GB = 400GB + 124GB + 500GB).

Provisioned capacity:
  The amount of capacity that would be used in the storage array's pool being
  used by Cinder if all the volumes present in there were completely full.

  This is currently being reported by the drivers as `provisioned_capacity_gb`
  and, as the name indicates, should be reported in GB and with a precision no
  greater than 2 decimals.  This is a required field and *must always be
  present*.

  This includes not only volumes created by Cinder but also all other existing
  volumes in that backend, but *does not include snapshots*.

  Let's expand the earlier example from "free capacity" where 524GB of the
  available 1TB had already been used, and say that the 124GB that were
  externally created were all used by 1GB thick volumes, and that Cinder was
  using the 400GB with 400 thick volumes of 1GB and 20 empty thin volumes of
  20GB each.  In this situation our reported `provisioned_capacity_gb` value
  should be 924GB ((124 * 1GB) + (400 * 1GB) + (20 * 20GB)).

  If a driver does not report the `provisioned_capacity_gb` data we'll use the
  automatically calculated `allocated_capacity_gb` as described below.

Allocated capacity:
  Contrary to what the name may suggest this is not referring to the
  "allocated" space on the storage array, but to the provisioned volumes
  created by this specific Cinder Volume backend process on the storage array's
  pool being used by Cinder and that still present.

  Important to notice that this refers to a specific service backend, so if you
  are running a multi-backend Cinder service or multiple Cinder Volume services
  where you have more than one backend configured to use the same storage
  array's pool, then each one of these backends will only be reporting the
  sum of the `volume_size` of the volumes they created and not the sum of all
  the `volume_size` of the volumes that have been created by a Cinder service.

  This is currently being reported by the Volume service as
  `allocated_capacity_gb` and, as the name indicates, should be reported in GB.

  For two volumes had been created, one thick and one thin, each one of 1GB,
  then you'll be reporting 2GB as `allocated_capacity_gb`, but if you were to
  unmanage one of those volumes then you would only be reporting 1GB, even if
  the volume is still there and will still be counted in the
  `provisioned_capacity_gb`.

  This field is calculated directly by the Cinder core code and drivers should
  not calculate or report this information on their `get_volume_stats` method.

Over subscription ratio:
  It is the maximum ratio between the "provisioned capacity" and the "total
  capacity" represented as a real number.  A ratio of 1.0 means that the
  "provisioned capacity" cannot exceed the "total capacity" whereas a value of
  5.0 means that the Cinder backend is allowed to create as much as 5 times the
  "total capacity" of the storage array's pool in volumes.

  This will only have effect when a thin provisioned volume is being created,
  and will be ignored for thick provisioned.

  This is currently being reported by the drivers as
  `max_over_subscription_ratio` with a greater or equal value to 1.0,
  preferably with no more than a 2 decimal precision.

  This value is optional, and when missing from the driver's status report the
  value defined in the `[DEFAULT]` section on the Cinder scheduler receiving
  the request will be used.  So vendors should make sure that they are
  correctly returning this value in their drivers if they support thin
  provisioning and admins should make sure they have a consistent default value
  of the `max_over_subscription_ratio` across all scheduler nodes.

  Note that this ratio is per backend or per pool depending on driver
  implementation.

Reserved percentage:
  Represents the percentage of the storage array's "total capacity" that is
  reserved and should not be used for calculations.  It is represented by an
  integer value going from 0 up to 100.

  This is currently being reported by the drivers as
  `reserved_percentage` with a greater or equal value to 1.0, preferably
  with no more than a 2 decimal precision.

  Default value is 0 if the field is missing in the status report from the
  backend or if the user has not defined it in the backend's Cinder
  configuration.  This is per backend or per pool depending on driver
  implementation.

Provisioning support:
  Cinder backends may support up to two different types of provisioning, *thin*
  and *thick* and drivers are expected to indicate as capable of one of them at
  least in their capabilities report.

  The way to report support for these is setting to true the boolean fields
  `thin_provisioning_support` and/or `thick_provisioning_support`.  And non
  reported provisioning types will default to false.

  A Cinder backend may support both provisioning types at the same time.

Volume provisioning type:
  For Cinder backends that only support one of the provisioning types all
  volumes created on them will be of that type, and we can use the volume
  type's extra specs to make the scheduler filter out backends not supporting a
  specific provisioning type:

  - 'thin_provisioning_support': '<is> True' or '<is> False'
  - 'thick_provisioning_support': '<is> True' or '<is> False'

  But if our deployment is using a backend that is supporting both provisioning
  types simultaneously we need to be explicit about the type of provisioning we
  want for a volume using the volume type's extra spec `provisioning:type` and
  setting it to `thin` or `thick`.

  If no `provisioning:type` is defined for a volume it will default to thin if
  the backend is capable of it, and the driver is expected to honor this
  assumption.

Incorrect reports
-----------------

Given above terms which were originally defined in their corresponding specs,
even if there may be additional comments in this one, we can determine that
there are a good number of Cinder drivers that do not follow these definitions
and are reporting what would be incorrect values.

Reporting incorrect values means that on a heterogeneous cloud you'll have
inconsistent scheduling and an admin will not be able to make sense of the
stats from the volumes.

To illustrate this here are some of the interpretations we can see across
different drivers for the `provisioned_capacity_gb`:

* Sum of all the volumes' max sizes, which is correct.
* Sum of all the volumes' physical disk usage, which is wrong.
* Sum of the Cinder volumes' physical disk usage, which is wrong.

And something similar happens with the `allocated_capacity_gb` where drivers go
and report the value directly instead of letting the Cinder core code take care
of it.  Drivers have been known to report here the following information:

* Sum of the Cinder volumes' physical disk usage, which is correct.
* Sum of the physical disk usage, which is wrong.
* Sum of all the volumes' max sizes, which is wrong.

Provisioning calculations
-------------------------

Some of the creation failures are based on the `provisioned_capacity_gb` value
being wrong, but there are other cases where Cinder's calculations for over
provisioning do not match industry's standard definition, which for some admins
create confusion and undesired behavior.

Standard provisioning calculation to check if a volume of `volume_size` fits
is::

  ((provisioned_capacity_gb + volume_size) <=
   (total_capacity_gb
    x (1 - (reserved_percentage / 100.0))
    x max_over_subscription_ratio))

Whereas the Cinder calculations, which were agreed on as the best calculations
for being considered safer are::

  (volume_size <=
   (free_capacity_gb
    - (total_capacity_gb x reserved_percentage / 100.0))
   x max_over_subscription_ratio)


Calculating max over subscription ratio
---------------------------------------

Most deployments have very dynamic workloads each with different physical
storage requirements, which means that one month we may require many volumes
of which we barely use any space and next month we may require fewer volumes
but use most of the provisioned capacity.

This makes it almost impossible to accurately model our storage requirements
at deployment time, which is precisely when we have to set the
`max_over_subscription_ratio` for our Cinder backends.

As requirements change one option would be to change the configuration and
restart our Cinder Volume services, but since Cinder is also in the data path
the restart may take a long time to do and will have a considerable impact on
our cloud users.

Not being able to determine beforehand the best `max_over_subscription_ratio`
and not being able to easily restart the Cinder service is a common pain that
most operators have with backends supporting thin provisioning.

Use Cases
=========

The basic case for fixing the status report is where we would like to have
consistent reporting from our backends for the admins to see in the logs and
for the scheduler to use.

Any operator using thin provisioning storage that wants to optimize their
storage usage and dynamically adjust to the dynamic requirements of its cloud.

As for the alternative calculations it would greatly benefit any backend that
is close to their full capacity or one that is creating huge volumes that
usually never get filled in.

Proposed change
===============

Incorrect reports
-----------------

Since we have consolidated all the documentation in one place, this spec, where
we clearly state expected driver behavior all driver maintainers will be urged
to make their drivers compliant with it.  This will mean adapt their drivers
to follow this document's definition for `provisioned_capacity_gb` and stop
reporting the `allocated_capacity_gb` field.

Automatic over subscription ratio calculation
---------------------------------------------

To allow automatic over subscription ratio calculation we will add a new
configuration option named `auto_max_over_subscription_ratio` that will
instruct Cinder to use configured `max_over_subscription_ratio` as a starting
reference when the backend is empty and then, when there is data calculate the
current value on each driver stats report with the following formula::

  adjusted_total = `total_capacity_gb` x (1 - (`reserved_percentage` / 100.0))
  ratio = `provisioned_capacity_gb` / adjusted_total - `free_capacity_gb`

If the driver is not reporting `provisioned_capacity_gb` then we'll proceed to
use the `allocated_capacity_gb` instead::

  adjusted_total = `total_capacity_gb` x (1 - (`reserved_percentage` / 100.0))
  ratio = `allocated_capacity_gb` / adjusted_total - `free_capacity_gb`

This new configuration option will be independent of the drivers and it will be
part of Cinder's core code, so if `auto_max_over_subscription_ratio` is not
defined or set to `False` then Cinder will continue behaving as it is now
(returning the `max_over_subscription_ratio` that is reported by the driver or
the one configured by default if not present).  But if it is set to True it
will always return calculated ratio as explained.

There are a couple of drivers that are already doing this, Pure and Kaminario's
K2, but with different configuration options,
`pure_automatic_max_oversubscription_ratio` and
`auto_calc_max_oversubscription_ratio` respectively, so we'll deprecate those
configuration options and remove the code within those drivers when we add the
generic code to Cinder.

Provisioning calculations
-------------------------

Instead of keep fighting with admins and developers on which one of the
approaches is best -standard calculation or Cinder's- we will be adding a new
configuration option called `over_provisioning_calculation` which will take
values `standard` and `cinder` and default to `cinder` for backward
compatibility and that will be used by the `CapacityFilter` to determine which
one of the mechanism to use.

This configuration option will also affect `CapacityWeigher` as it will need to
do the free space calculation according to the standard definition as well.

As one can assume thick provisioning will have no modifications on its
behavior.

Alternatives
------------

* Don't support standard over-provisioning calculations.
* Instead of modifying `CapacityFilter` and `CapacityWeigher` create 2 new
  classes.
* Instead of adding the `over_provisioning_calculation` configuration option
  make the filter use the options JSON file provided by
  `scheduler_json_config_location` .  This data seems to be currently missing
  on some of the operations like migrate, extend, so that would need to
  changed.

Data model impact
-----------------

N/A

REST API impact
---------------

The only affected API will be the `get_pools` API that will be able to return
the 2 new fields, `total_used_capacity_gb` and `cinder_used_capacity_gb`, when
they are being reported by the driver's `get_volume_stats` method.  Fields will
not be present if the drivers are not reporting them.

Security impact
---------------

N/A

Notifications impact
--------------------

N/A

Other end user impact
---------------------

The user may see new fields when calling cinderclient's `get_pools`.

Performance Impact
------------------

Depending on the driver and the storage array, performance could increase or
decrease, since getting provisioned sizes instead of physical sizes could be
faster or slower.

Other deployer impact
---------------------

With the change of values returned by Cinder backends for
`allocated_capacity_gb` and `provisioned_capacity_gb` we may experience
failures on creating volumes until we correct the values of
`reserved_percentage` and `max_over_subscription_ratio` in our cloud to the
right values, since we may have been using incorrect ones.

Two new configuration options will be added:

* `auto_max_over_subscription_ratio`: Boolean value that will instruct Cinder
  to automatically calculate the over subscription ratio based on current usage
  instead of using a fixed value.

* `over_provisioning_calculation`: Will allow to select what kind of
  calculations the `CapacityFilter` does to determine if there is space for a
  volume in a backend.  Acceptable values are `standard` and `cinder`.  Default
  values will be `cinder`.

Developer impact
----------------

Driver maintainer will need to verify, and fix if necessary, their stat reports
for `allocated_capacity_gb` and `provisioned_capacity_gb` unless they start
using the new `auto_max_over_subscription_ratio` configuration option.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  None

Other contributors:
  None

Work Items
----------

* File bugs for drivers that are not in compliance.
* Fix drivers stat reporting.
* Add support for the 3 new fields, `provisioned_capacity_precission`,
  `total_used_capacity_gb` and `cinder_used_capacity_gb` in the scheduler, the
  `get_pools` API, and the client.
* Modify `CapacityFilter` to support the standard over-provisioning
  calculation.
* Modify `CapacityWeigher` to support standard over-provisioning calculations.
* Add to the volume manager the estimation mechanism for drivers that don't
  report `provisioned_capacity_gb`.
* Update all the developers reference docs to ensure that there is no more
  confusion on what the report stats need to return, and make sure that the
  wiki page on how to contribute a driver links to that documentation
  explaining the importance of following it when writing the driver.

Dependencies
============

N/A

Testing
=======

New unit tests will be added to test the changed code.

Documentation Impact
====================

Since our current documentation is lacking in this aspect this will add and
update it to reflect what's expected of the driver in the stats reports.

End user documentation should also be updated.

References
==========

.. [1] https://specs.openstack.org/openstack/cinder-specs/specs/kilo/over-subscription-in-thin-provisioning.html
.. [2] https://specs.openstack.org/openstack/cinder-specs/specs/newton/differentiate-thick-thin-in-scheduler.html
.. [3] https://specs.openstack.org/openstack/cinder-specs/specs/liberty/standard-capabilities.html

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================================
Filtering and weighing backends with driver supplied functions
==============================================================

https://blueprints.launchpad.net/cinder/+spec/filtering-weighing-with-driver-supplied-functions

Cinder currently uses a filter scheduler to filter out unqualified backends
during volume operations.  The default filters and weigher do a good job of
filtering out the unqualified backends but there are occasions when they work
less than ideally.  For example, due to thin-provisioning or other advanced
data manipulation mechanisms backends may have, properties such as
"free capacity" will no longer be as useful when determing the best backend
to use.  A potential solution to this issue is to incorporate filtering and
weighing (goodness) functions that can be specified for each backend as part
of the scheduler decision making.


Problem description
===================

The default filters and weighers being used by the Cinder scheduler are not
determining the most qualified backends to use in some cases.  If a backend
is using an advanced data manipulation mechanism such as thin-provisioning the
"free capacity" property, which is used by the default filters, will not be as
useful or accurate when determining an ideal backend.

Another problem is that different backends use different metrics and only the
manufacturer really knows what they are.  This makes it hard/almost impossible
to create a global filter for these metrics.  An example of this would be if
there was a backend array with an absolute maximum capacity of 1000 volumes.
The performance of this array degrades if more than 75% of total space is
actually used.  There is also a maximum volume size of 500 GB.  These details
can be highly variable between vendors and even within different backend
arrays from the same vendor.


Proposed change
===============

The proposed solution for the above problem is to add two new properties to
the statistics that drivers support.  These two new properties will be a
filter and goodness function.  The details for the two new properties are
described here::

    * filter_function -- An equation in string form that when evaluated using
                         PyParsing will be True/False.  Used by the new filter
                         to determine if a backend should be considered by
                         the scheduler.

    * goodness_function -- An equation in string form that when evaluated using
                           PyParsing will be a value between 0 and 100.  Used
                           by the new filter to which valid backend is the best
                           for the scheduler to use.

An equation can be made up of any of the following operators as long as the
equation is valid:

============================ =======================
Operations                   Type
============================ =======================
+, -, \*, /, ^               standard math

not, and, or, &, \|, !       logic

>, >=, <, <=, =, ==, <>, !=  equality

+, -                         sign

x ? a : b                    ternary

abs(x), max(x, y), min(x, y) math helper functions
============================ =======================

New operators could be added in the future, if desired.  These ones should
cover the majority of equations an admin would need.

When the scheduler attempts to find the best host it gets the stats for each
backend's driver.  The backend(s) with the top goodness rating will be
returned to the scheduler.  There is a possibility of multiple backends because
there can be ties (three backends with goodness of 100).  Depending on the
weigher that an admin has chosen, such as capacity, random choice, etc, any
potential tie can be resolved and the winning backend will be chosen.

If a backend's driver does not return both filter_function and
goodness_function properties during stat reporting the new filter will default
to passing that backend and give it a goodness of 0.  When it comes time for
the schedueler to pick the best host a 0 would convert to a "use as a
last resort" rating.

If a driver chooses not to implement the new properties it will return the
standard stats during reporting with no changes.  Example::

    {
      "volume_backend_name": "array_one",
      "pools": {
          [
            "pool_name": "my_pool",
            "percent_capacity_used": 25,
            "free_space": 5000,
            "n_vols": 50,
            ...
          ],
          ...
      },
      ...
    }

Drivers that do choose to implement the new properties will need to report
the two new properties back during stat reporting.  An example is here::

    {
      "volume_backend_name": "array_one",
      "pools": {
          [
            "pool_name": "my_pool",
            "filter_function": "(stats.n_vols < 1000) and (volume.size < 5)",
            "goodness_function": "stats.percent_capacity_used < 75 ? 100 : 25",
            "percent_capacity_used": 50,
            "free_space": 1000,
            "n_vols": 100,
            ...
          ],
          ...
      },
      ...
    }

When a driver's get_volumes_stats method is called it is up to the driver to
determine how to generate the equations for both the filter and goodness
function properties.  Some choices a driver has are to use values defined in
cinder.conf, hard-code the values in the driver or not implement the
properties at all.  Drivers can implement other ways of achieving this, too, as
long as the requirements for the filter_function and goodness_function
properties are met.

Here is an example of an implementation of how using cinder.conf values
would work.  An admin can setup the filter and goodness function properties
for each backend.  Example::

    [foo_1]
    ...
    filter_function = "volume.size > 10 and volume.size <= 500"
    goodness_function = "50"

    [foo_2]
    ...
    filter_function = "volume.size > 500"
    goodness_function = "90"

The new filter will only focus on filtering based on the filter/weighing
functions provided by a driver.  If an admin desires filtering capabilities,
such as for capacity, the other scheduling filters available in Cinder can be
used along with this one.

In summary, implementing a new scheduling filter that allows for more
control over the filtering process with a filter and goodness function will
allow the best backend for a volume to be chosen correctly more often.  There
two potential downside to this solution.  First, the equations for the filter
and goodness functions are not validated until they are used by the scheduler.
If there is a typo or syntax error in either equation it will not be known
until the scheduler fails during evaluation.  In the future some form of
startup validation could be added to detect invalid equations.  A simpler
solution to implement for now would be to have the scheduler default to
assuming an invalid filter function is passing.  The goodness function would
have a default of 0 incase there is an invalid equation detected in the filter
or goodness functions.  The second downside is that the equations that
PyParsing will evaluate will require adequate documentation showing examples
of how the operators work, syntax , etc.  However, it should not be too hard
to document this.

Alternatives
------------

One alternative solution is to have two functions located in a file in a
known location (possibly the driver itself).  One function generates the
filter_function result and the other generates the goodness_function
result.  The backend driver would still return two new stat properties,
but they would now contain the import path for the needed functions.
The new filter will then import the needed functions and use them during
the filtering process.  There could also be default base functions incase
of a driver not implementing the new properties. Similar to the proposed
solution, a default of a pass and goodness of 0 would be given to those
backends using those drivers.  Selection of backends with tieing goodness
rating will be resolved in the same way as the proposed solution.

Example of custom defined filter and goodness functions::

    def filter_function(stats, volume, qos_specs, extra_specs, **kwargs):
        # Return True or False
        return bool(evaluate((stats.n_vols < 1000) and (volume.size < 5)))

    def goodness_function(stats, volume, qos_specs, extra_specs, **kwargs):
        # Return 0 to 100
        return clamp(evaluate(stats.percent_capacity_used < 75 ? 100 : 25))

The returned stats from a driver would now look like this::

    {
      "volume_backend_name": "array_one",
      "pools": {
          [
            "pool_name": "my_pool",
            "filter_function": "path.to.my.custom.module.filter_function",
            "goodness_function": "path.to.my.custom.module.goodness_function",
            "percent_capacity_used": 50,
            "free_space": 1000,
            "n_vols": 100,
            ...
          ],
          ...
      },
      ...
    }

Benefits of this solution are that the equations can be checked for typos and
syntax errors more easily since it will be Python code.  It would also be
possible to check for these errors on startup more easily than the proposed
solution.  Unit tests can be developed and maintained by an adminfor the
custom functions so that after an upgrade and an admin can ensure everything
is working still.

Another benefit is that Python is a well documented language.  The PyParsing
operators used by admins to create equations will have to have documentation
written to explain how to use the various operators.

A downside would be that admins would need to know how to work with python
functions.  Deciding how to maintain all the functions is another potential
downside depending on where they are placed.

Another downside to this solution is that the functions are not coming from
the driver directly.  They will not be updated by or with the driver if an
admin is the one creating them.  Values returned by a driver can potentially
change over time, e.g. a backend getting many small volumes can use the
goodness value to lean towards wanting bigger volumes.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Filtering and goodness function equations can be optionally set in
cinder.conf by an admin.  The equations for each function will be strings
that are parsed using pyparsing.  An admin can enter anything they want for
the string.  Pyparsing should be able to filter out and throw an exception
for any dangerous strings that an admin may enter.

Notifications impact
--------------------

None

Other end user impact
---------------------

Admins will be able to optionally set filtering and goodness function
equations in cinder.conf for backends if implemented by a driver.

Performance Impact
------------------

Depending on the complexity of a given filtering or goodness function
equation there may be a slight performance decrease when it is evaluated
with PyParsing.  Simple equations will have minimal performance impact.

Other deployer impact
---------------------

None

Developer impact
----------------

A backend's host stats, extra specs, volume info and qos specs will be
available to use in the filter and goodness functions.  Possibly more can be
exposed in the future.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  leeantho

Other contributors:
  kmartin

Work Items
----------

* Implement evaluator code using pyparsing.

* Implement a new driver filter for scheduler that will use the evaluator.

* Update driver code to return filter and goodness function properties during
  stat reporting for a backend.  This only needs to occur if a driver wants
  to implement support for this new filter.


Dependencies
============

The pyparsing module (2.0.1 or later).


Testing
=======

Unit tests can be used to validate the evaluator and filter code so new
Tempest tests are not needed.


Documentation Impact
====================

* Documentation will be needed to detail what driver properties are exposed
  for use in the filter/goodness functions.  Each driver would need to
  provide their own documentation for what properties they have.

* Documentation will be needed to explain the syntax for the filter and
  goodness function equations.  Samples of using each operator would be
  good, too.


References
==========

Pyparsing :: http://pyparsing.wikispaces.com/

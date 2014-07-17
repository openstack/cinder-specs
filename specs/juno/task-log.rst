..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Cinder task logging improvements
================================

https://blueprints.launchpad.net/cinder/+spec/task-logging

To improve the usefulness of the usage of tasks & flows in cinder in the
``create_volume`` workflow (with more workflows to come later) we need to
improve the level of awareness and usage of the interconnect
between `taskflow`_ and cinder. This will help track a single workflow while
it runs (allowing for introspection into what is occurring or what has
occurred). It will help for operators, for developers and others to have this
information available for usage.

.. _taskflow: http://docs.openstack.org/developer/taskflow/

Problem description
===================

Current taskflow usage does not take advantage of the taskflow `engine`_
internals event/notification mechanism. This makes it hard to determine
what is happening internally to taskflow and how to debug it when things
start to fail (which they inevitably do). It would be better to have this
information exposed in a more easier to use and understandable manner for
future use as well (but we first have to get the basics working in the
first place).

.. _engine: http://docs.openstack.org/developer/taskflow/engines.html

Proposed change
===============

To start we need to hook into the `notification`_ system that taskflow engines
provide/emit. This will solve the issue of having a useful set of information
being emitted by the taskflow engines used in cinder and will help in
debugging the state-transitions & activity of tasks and flows ran there-in.
This will help solve the issue of debuggability and tracking of the internals
of taskflow; making the lives of operators, developers, and users much easier.

Part of the proposed change and one that needs more feedback is the location of
this log (is it a log at all?). Using the `log listener`_ that taskflow
provides we can plug in any type of logging compatible object (typically
a ``LOG`` from the python logging module) as the receiver of these events. In
the cinder case this should be a oslo incubator ``LOG`` object (as is typically
used in cinder and elsewhere in openstack). The question becomes where should
this ``LOG`` output the task and flow events to.

The following suggestion seems to be agreed upon:

* A log location for task/flow events, provided by a new configuration
  option ``task_log_file`` (the configuration option name can be changed as
  needed, this was just a example and may not be the best name for this usage)
  that would be set in ``cinder.conf`` that would specify where this data would
  be written to. If this location is not set then it will default to the main
  cinder log (this will allow the log to be separated out for those that
  desire to do this).

.. _notification: http://docs.openstack.org/developer/taskflow/notifications.html
.. _log listener: http://docs.openstack.org/developer/taskflow/notifications.html#printing-and-logging-listeners

Alternatives
------------

N/A

Data model impact
-----------------

N/A

REST API impact
---------------

Not currently.

Some ideas that could be useful in the future:

* Provide the same information (or a reduced set of this information) to end
  users of the cinder api so that they can gain insight into what is occurring
  with there request inside cinder. This would involve saving the same event
  stream into something not a log file but some more appropriate structure that
  can be given back to the user in a nice format (json or other). This is
  similar to providing a *task log* back to the user of cinder that other
  OpenStack projects are also currently exploring/implementing.

Security impact
---------------

N/A

Notifications impact
--------------------

N/A

Other end user impact
---------------------

N/A

Performance Impact
------------------

More log data would mean a rotation mechanism would need to be setup if it was
not setup previously. Since these log files can contain a lot of information
they would need to be rotated at a faster rate than log files that only
contain ``ERROR`` or ``WARNING`` information.

Other deployer impact
---------------------

An optional configuration setting to enable a new log file location. If a
log file location is added then it would involve a new log file rotation and
associated policy and this is something operators would need to be aware of
and correctly configure to avoid filling up a servers hard drive. The goal is
that this new configuration option would default to the existing main cinder
log (so that this impact would be minimal for most operators).

Developer impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee: harlowja

Work Items
----------

* Create new config options for cinder for event stream target location.
* Use event stream in ``create_volume`` usage of `taskflow`_ task/flows and
  engines.

Dependencies
============

N/A

Testing
=======

It should be feasible to replace the ``LOG`` or notification mechanism being
used in cinder with taskflow with a ``mock`` object that can also gather the
same event stream and allow it to be used in a test to verify that the event
stream contains the desired events (likely we should be selective in the test
and only verify that certain *key* events were emitted, since taskflow could
add new events in the future).

Documentation Impact
====================

N/A

References
==========

Summit discussion:

* https://etherpad.openstack.org/p/juno-cinder-state-and-workflow-management

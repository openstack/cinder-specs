..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Rootwrap daemon mode
====================

https://blueprints.launchpad.net/cinder/+spec/rootwrap-daemon-mode

Cinder is one of projects that require root privileges. Currently this
is achieved with oslo.rootwrap that has to be run with sudo. Both sudo
and rootwrap produce significant performance overhead. This blueprint
is one of the series of blueprints that would cover mitigating rootwrap
part of the overhead using new mode of operations for rootwrap - daemon
mode. These blueprints will be created in several projects starting
with oslo.rootwrap [#rw_bp].

Problem description
===================

As you can see in [#ne_ml] rootwrap presents big performance overhead for
Neutron. Impact on Cinder is not as significant but it is still there.
Details of the overhead are covered in [#rw_bp].

Use Cases
=========
This will eliminate bottleneck in large number of concurrent executed
operations.

Proposed change
===============

This blueprint proposes implement changes that allow to run oslo.rootwrap
daemon. The daemon works just as a usual rootwrap but accepts commands to
be run over authenticated UNIX domain socket instead of command line and
run continuously in background.

Note that this is not usual RPC over some message queue. It uses UNIX socket,
so no remote connections are available. It also uses digest authentication
with key shared over stdout (pipe) with parent process, so no other processes
will have access to the daemon. Further details of rootwrap daemon are covered
in [#rw_bp].

``use_rootwrap_daemon`` configuration option should be added that will make
``utils.execute`` use daemon instead of usual rootwrap.

Alternatives
------------

Alternative approaches have been discussed in [#rw_eth].

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

This change requires additional endpoint to be available to run as root -
``cinder-rootwrap-daemon``.

All security issues with using client+daemon instead of plain rootwrap are
covered in [#rw_bp].

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

This change introduces performance boost for disk operations that are
required to be run with root privileges. Current state of rootwrap daemon
in Neutron shows over 10x speedup comparing to usual ``sudo rootwrap`` call.
Total speedup for Cinder shows impressive results too [#rw_perf]:
test scenario CinderVolumes.create_and_delete_volume
Current performance :
+----------------------+-----------+-----------+-----------+-------+
| action               | min (sec) | avg (sec) | max (sec) | count |
+----------------------+-----------+-----------+-----------+-------+
| cinder.create_volume | 2.779     | 5.76      | 14.375    | 8     |
| cinder.delete_volume | 13.535    | 24.958    | 32.959    | 8     |
| total                | 16.314    | 30.718    | 35.96     | 8     |
+----------------------+-----------+-----------+-----------+-------+
Load duration: 131.423681974
Full duration: 135.794852018

With use_rootwrap_daemon enabled:

+----------------------+-----------+-----------+-----------+-------+
| action               | min (sec) | avg (sec) | max (sec) | count |
+----------------------+-----------+-----------+-----------+-------+
| cinder.create_volume | 2.49      | 2.619     | 3.086     | 8     |
| cinder.delete_volume | 2.183     | 2.226     | 2.353     | 8     |
| total                | 4.673     | 4.845     | 5.3       | 8     |
+----------------------+-----------+-----------+-----------+-------+
Load duration: 19.7548749447
Full duration: 22.2729279995


Other deployer impact
---------------------

This change introduces new config variable ``use_rootwrap_daemon`` that
switches on new behavior. Note that by default ``use_rootwrap_daemon`` will be
turned off so to get the speedup one will have to turn it on. With it
turned on ``cinder-rootwrap-daemon`` is used to run commands that require root
privileges.

This change also introduces new binary ``cinder-rootwrap-daemon`` that should
be deployed beside ``cinder-rootwrap``.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Anton Arefiev(aarefiev)

Work Items
----------

The only work item here is to implement new config variable and run rootwrap
in daemon mode with it.

Dependencies
============

* rootwrap-daemon-mode blueprint in oslo.rootwrap [#rw_bp]_.

Testing
=======

Rootwrap has it's own functional testing for the rootwrap client/daemon
pieces [#rw_func]_, Cinder part will be covered by unit tests.

Cinder has an unusual usecase where tens/hundreds of mb are passed over
stdin/out (sheepdog backup) - that test case should be covered in the
functional tests.

Also we can add new Tempest job with turned on use rootwrap daemon flag.

Documentation Impact
====================

Set ``use_rootwrap_daemon=True`` configuration option in cinder.conf to make
``utils.execute`` use daemon instead of usual rootwrap.

References
==========

.. [#rw_bp] oslo.rootwrap blueprint:
   https://blueprints.launchpad.net/oslo.rootwrap/+spec/rootwrap-daemon-mode

.. [#ne_ml] Original mailing list thread:
   http://lists.openstack.org/pipermail/openstack-dev/2014-March/029017.html

.. [#rw_func] Rootwrap daemon functional testing
   https://github.com/openstack/oslo.rootwrap/blob/master/tests/test_functional.py

.. [#rw_perf] Cinder performance testing results
   http://paste.openstack.org/show/160890/

.. [#rw_eth] Alternative approaches
   https://etherpad.openstack.org/p/neutron-agent-exec-performance

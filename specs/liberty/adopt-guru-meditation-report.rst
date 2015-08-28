..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Adopt Guru Meditation report
============================

https://blueprints.launchpad.net/cinder/+spec/guru-meditation-report

This spec proposes adoption for Oslo Guru Meditation reports in Cinder.
The new feature will enhance debugging capabilities of all official Cinder
services, by providing an easy way to gather runtime data about current
threads and configuration, among other things, to developers, operators,
and tech support.


Problem description
===================

Currently Cinder does not have a way to gather runtime data from active
service processes. The only information that is available to deployers,
developers, and tech support to analyze is what was actually logged by the
service in question. Additional data could be usefully used to debug and solve
problems that occur during Cinder operation. Among other things, we could be
interested in stack traces of all (both green and real) threads, pid/ppid info,
package version, configuration as seen by the service, etc.

Oslo Guru reports provide an easy way to add support for gathering this kind of
information to any service. Report generation is triggered by sending a special
(USR1) signal to a service. Reports are generated on stderr, that can be piped
into system log, if needed.

Use Cases
=========

Guru reports are extensible, meaning that we will be able to add more
information to those reports in case we see it needed.

Guru reports has been supported by Nova.


Proposed change
===============

First, a new oslo-incubator module (reports.*) should be synchronized into
cinder tree. Then, each service entry point should be extended to register
reporting functionality before proceeding to its real main().


Data model impact
-----------------
None.


REST API impact
---------------
None.


Security impact
---------------
In theory, the change could expose service internals to someone who is able to
send the needed signal to a service. That said, we can probably assume that the
user is already authorized to achieve a lot more than just having an access to
stack traces and configuration used. Also, if deployers are afraid of the
information leak for some reason, they could also make sure their stderr output
is channeled into safe place.

Because this report is triggered by user, there is no need to add config option
to turn on/off this feature.

Notifications impact
--------------------
None.


Other end user impact
---------------------
None.


Performance Impact
------------------
The feature does not require any additional resources until it's triggered by
the user. Default report generation is not expected to take too long. Report
extensions will need to be assessed on case by case basis. In any case, reports
are not expected to be generated too often and are assumed to be debugging
capability, not something to trigger once per minute just in case.


IPv6 Impact
-----------
None.


Other deployer impact
---------------------
Deployers may be interested in making sure those reports are collected
somewhere (e.g. stderr should be captured by syslog).


Developer impact
----------------
None.


Community Impact
----------------
None.


Alternatives
------------
We could reimplement the wheel, but we hopefully won't.


Implementation
==============

Assignee(s)
-----------
wanghao


Work Items
----------
* sync reports.* module from oslo-incubator
* adopt it in all cinder services using a wrapper located under
  cinder/cmd/...


Dependencies
============
reports.* module is currently going thru graduation consideration. In case it's
graduated into oslo.reports library before cinder switches to it, we won't
actually need to sync any code from oslo-incubator but instead add a new
external oslo dependency. If Cinder switches to the module before graduation
is complete, then we'll need to adopt oslo.reports later as part of usual oslo
liaison effort.


Testing
=======
Automated tests are needed to ensure the guru report is working correctly.


Documentation Impact
====================
User documentation will need to be updated to introduce this changes.

Developer documentation should be updated to include information on how to add
support for the reporting feature.


References
==========
* oslo-incubator module: http://git.openstack.org/cgit/openstack/oslo-incubator/tree/openstack/commo
n/report
* blog about nova guru reports: https://www.berrange.com/posts/2015/02/19/nova-and-its-use-of-olso-i
ncubator-guru-meditation-reports/
* oslo.reports repo: https://github.com/directxman12/oslo.reports
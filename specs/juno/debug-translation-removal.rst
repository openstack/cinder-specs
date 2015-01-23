..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Remove Translation from Debug Messages
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/debug-translation-removal

Including translation of debug messages for OpenStack creates more
work for the translators than can be contained.  It has been decided
to reduce the workload for translators and to ensure that the most
important log messages are translated properly, we remove
translation of debug messages.

Problem description
===================

We have currently been doing Cinder development under the assumption
that all messages should be translated.  As described above, it has
been decided that this is not the case.  In order to bring Cinder in
line with other OpenStack components we need to remove translation
from debug level log messages.

Use Cases
=========

Proposed change
===============

The change proposed is simply to remove _() from around any debug
level log messages.  While the change is simple, the changes touch
many, if not all, files in Cinder which requires careful coordination
of the implementation of the change.

The proposed implementation of this change is to have volume driver
owners take responsibility to implement changes in each of the
directories, (cinder/volume/drivers/*) to start with, as individual
commits.  We will then work backwards and split up making changes
across the remaining top level directories (cinder/*).  Each TLD will
be handled as its own commit.

Note that this spec and associated blueprint are not to address the
integration of _LI(), _LW(), _LE() and _LC() functions.  That work is
being addressed by the following spec/blueprint:
https://review.openstack.org/#/c/99853/

Alternatives
------------

We could not go back and remove  from existing log messages.  This,
however, seems like a poor decision given that we would not be consistent
with other OpenStack projects and it would be confusing and ugly as we
go forward given that some debug messages would have  around them while
others would not.

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

End users will no longer be able to see debug output in locale's
aside from English.  This, however, was deemed necessary to allow
the translation team to be able to focus on translation of other
log levels.

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployers who previously have seen debug messages translated will
need to be aware that this change has been implemented.

Developer impact
----------------

Developers should no longer translate debug level messages.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jsbryant (jsbryant@us.ibm.com and jungleboyj on IRC)

Other contributors:
  There will be numerous other contributors to this Blueprint
  as it is anticipated that the work will be split across
  multiple members of the Cinder development team.

Work Items
----------

Remove _() from all debug messages in Cinder.
Add a HACKING check to ensure that new cases of this are not added.


Dependencies
============

None


Testing
=======

Unit test for this change is not required.  A HACKING check, however,
should be added to test for developers attempting to translate debug
messages in the future.  The check should make sure that any LOG.debug
messages do not have a _() around the text contained in the message.


Documentation Impact
====================

The fact that debug messages shouldn't be translated is already
documented here:  https://wiki.openstack.org/wiki/LoggingStandards
Further documentation should be required.


References
==========
* Logging Standards documentation:
  https://wiki.openstack.org/wiki/LoggingStandards
* This issue was discussed in the 6/11/14 Weekly Cinder Meeting:
  http://eavesdrop.openstack.org/meetings/cinder/2014/cinder.2014-06-11-16.03.log.html


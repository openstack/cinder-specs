..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
i18n Enablement for Cinder
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/i18n-enablement

This BluePrint/Spec proposes completing the enablement of i18n
(internationalization) support for Cinder.

Internationalization implementation has been an on-going effort in OpenStack
during recent releases.  During the Icehouse release, much of the support
for internationalization was already merged into Cinder.  Specifically
the update of Oslo's gettextutils (commit
1553a1e78ec262b044ce99b418103c91b7b580f6) completed much of
the process.  Removal of the use of str() in exceptions and messages
was the other major piece of work that was implemented: (commit
cbe1d5f5e22e5f792128643e4cdd6afb2ff2b5bf).

To finalize this work in Juno we need to enable "lazy" translation.
Enablement of lazy translation will allow end users to not only have
logs produced in multiple languages, but adds the ability for REST
API messages to also be returned in the language chosen by the user.
This functionality is important to support the use of OpenStack by the
international community.


Problem description
===================

Currently, Cinder does not have the all the code in place to support
lazy translation.  The code associated with this blueprint will add
the appropriate code and enable translation of REST API responses.

Proposed change
===============

The code for this change will add 'gettextutils.enable_lazy() to each of
the binaries in bin.

It will also remove the use of gettextutils.install() in each of the
binary files.  Instead it will add the explicit import of _() in all
files that are not already importing the _() function.  The need for
the change to explicitly import _() is documented
in bug https://bugs.launchpad.net/cinder/+bug/1306275 .

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

There is no additional changes to the REST API other than the fact
that the change enables the customer to specify the language they
wish REST API responses to be returned in using the Accept-Language
option.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Once merged this feature is immediately available to users.


Developer impact
----------------

The developer impacts have already been in place for some time.  Developers
have been using _() around messages that need translation.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <jsbryant@us.ibm.com> (Jungleboyj)

Other contributors:
  <jecarey@us.ibm.com>

Work Items
----------

I am planning to implement this as two patches.  The first will be the
patch to ensure that _() is being explicitly imported.  The dependent
patch will then set enable_lazy().


Dependencies
============

None.


Testing
=======

There will be a tempest test added for Cinder that will ensure that
lazy translation is working properly.


Documentation Impact
====================

None.


References
==========

None.

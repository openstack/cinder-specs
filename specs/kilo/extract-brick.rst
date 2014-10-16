..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Extract Brick library from Cinder
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/extract-brick

This Specification talks about 2 points.  First the extraction
of the brick/cinder directory into it's own standalone library.
Second, changing cinder to use the external library.

Problem description
===================

The idea of 'brick' was created back in the Havana timeframe.  All along
it was meant to be a standalone library that Cinder, Nova and any other
project in OpenStack could use.  Currently brick lives in a directory
inside of Cinder, which means that only Cinder can use it.


Proposed change
===============

We want to extract the brick directory and encapsulate it into it's own pypi
library that any python project can use.

So we need to do:

* First create a separate python project and release it into pypi
* Add brick to cinder's requirements.txt
* Remove the existing cinder/brick directory
* Everywhere in Cinder that uses cinder/brick will need to be modified to use
  the new pypi library.

Alternatives
------------

We could simply keep brick inside of Cinder and not share it's code.  The
problem with this is that any changes/fixes to brick will then need to be
backported into the same code in Nova.   This is the existing problem.

Data model impact
-----------------

This doesn't change the data model for Cinder.

REST API impact
---------------

None

Security impact
---------------

Any security related issues that impact brick may impact anything in Cinder
that would use the new library.   Brick will be placed into stackforge for
code reviews, so anyone can contribute fixes.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

There should be no performance impact, as the same code will exist in the
pypi brick library as it does today in Cinder.   The import will pull the
library from a system installed location instead of the cinder codebase.


Other deployer impact
---------------------

Brick will be a requirement for Cinder.  So whoever builds distribution
packages for cinder will also need to ensure that brick gets installed.


Developer impact
----------------

Currently, any Cinder developer can add new features and fix bugs in brick
as part of the Cinder project.   Going forward, developers will need to clone
the source of brick from stackforge and make their changes there.

The downside to this is that If there are new features, APIs or interface
changes in the brick library, a release will have to be published in order
for Cinder to get those changes.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Walter A. Boring IV - walter-boring

Other contributors:
  e0ne

Work Items
----------

I have already started the work of creating a standalone brick library here:
https://github.com/hemna/cinder-brick

* This repo will need to be registered into stackforge, so brick can benefit
  from the community process and gerrit.
* Once in stackforge, an official release needs to be made so that the package
  exists in pypi.
* Then a patch to Cinder can be made to add it to the requirements.txt
* The another patch to follow that removes brick/cinder and adjusts all of
  Cinder to use the new library.


Dependencies
============

None

Testing
=======

All of the existing unit tests in cinder for brick can remain to ensure that it
works as advertised.   The existing Cinder brick unit tests will be migrated to
the new project and run there as well.


Documentation Impact
====================

None.


References
==========

* Existing standalone project: https://github.com/hemna/cinder-brick
* The original Cinder brick proposal
  https://wiki.openstack.org/wiki/CinderBrick

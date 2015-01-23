..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Affinity and anti-affinity scheduler filter
===========================================

https://blueprints.launchpad.net/cinder/+spec/affinity-antiaffinity-filter

To add scheduler filter to Cinder that allows scheduler to make placement
decision based on affinity relationship between existing volumes and new
volume (the one being scheduled).  The affinity relationship here means
the location of volumes ('host' of volume).

Problem description
===================

Cinder has done a good job hiding the details of storage backends from end
users by using volume types.  However there are use cases where users who
build their application on top of volumes would like to be able to 'choose'
where a volume be created on.  How can Cinder provide such capability without
hurting the simplicity we have been keeping?  Affinity/anti-affinity is one
of the flexibility we can provide without exposing details to backends.

The term affinity/anti-affinity here is to to describe the relationship
between two sets of volumes in terms of location.  To limit the scope, we
describe one volume is affinity with the other one only when they reside in
the same volume back-end (this notion can be extended to volume pools if
volume pool support lands in Cinder); on the contrary, 'anti-affinity'
relation between two sets of volumes simply implies they are on different
Cinder back-ends (pools).

This affinity/anti-affinity filter filters Cinder backend based on hint
specified by end user.  The hint expresses the affinity or anti-affinity
relation between new volumes and existing volume(s).  This allows end
users to provide hints like 'please put this volume to a place that is
different from where Volume-XYZ resides in'.  Below are two use cases where
this new filter can be useful.

1) DB team builds MySQL master onto one volume, they'd prefer to put new
   volumes for slave DBs to different backends from where the master DB
   resides in, for the sake of high availability.
2) Log processing project would like to have fast storage as possible, so
   they create soft RAID across multiple volumes. They want to put these
   volumes as close to each other as possible, ideally on the same storage
   backend, for the sake of performance.

Use Cases
=========

Proposed change
===============

Add two new filters to Cinder - AffinityFilter and AntiAffinityFilter.  These
two filters will look at the scheduler hint provided by end users (via
scheduler hint extension) and filter backends by checking the 'host' of
old and new volumes see if a backend meets the requirement (being on the same
backend as existing volume or not being on the same backend(s) as existing
volume(s)).


Alternatives
------------

There had been one proposal to allow admin user to directly specify the
backend for new volumes.  It doesn't really provide similar functionality as
affinity filter 'cos it was admin only and it itself has a few drawbacks
(security concern, for example).

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Although this change involves using or parsing user-provided - scheduler hints,
which is already part of Cinder.  This doesn't put Cinder in any more danger
as it is now.


Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

New filters would query DB once per request, it only adds slightly latency
to the system and the latency has nothing to do with the size of the system.


Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zhiteng-huang (winston-d on IRC)

Work Items
----------

1. Filter implementation
2. Internal DB API modifition


Dependencies
============

None

Testing
=======

Test against AffinityFilter:
 * Create one volume A;
 * Create another volume B with uuid of A and 'same_backend' as hint;
 * Checks if B is created on same backend as A;

Test against AntiAffinityFilter:
 * Create one volume A;
 * Create another volume C with uuid of A and 'different_backend' as hint;
 * Checks if C is created on different backend as A;

Documentation Impact
====================

Need to document the usage of new filters.


References
==========

Nova has been offering simliar feature called SameHostFilter and
DifferentHostFilter since *Diablo*.

https://github.com/openstack/nova/blob/master/nova/scheduler/filters/affinity_filter.py

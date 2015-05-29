..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Standard Capabilities
=====================

https://blueprints.launchpad.net/cinder/+spec/standard-capabilities

Problem description
===================

For the Liberty release there is a proposal [1] to allow storage backends to
push capabilities of their pools [1]. Eventually we would want some common
capabilities to graduate into becoming a ``well defined`` capability. The
point of this spec is to agree on the initial ``well defined`` capabilities.


Proposed change
===============

The initial ``well defined`` capabilities are:

* QOS
* Compression
* Replication
* Thin provisioning

Keep in mind this is just an agreement that these are common features that
a backend could support. As shown in the proposal of how this will work [1],
backends will still be able to push up the specific keys they look for in
volume types for these capabilities.

Alternatives
------------

None

Use Cases
---------

Having the ``well defined`` capabilities will allow the deployer to see what
common capabilities are shared beyond their deployed backends in Cinder.

Data model impact
-----------------

None. This information comes directly from the volume drivers, reported to the
scheduler, to the Cinder API.

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

None

Other deployer impact
---------------------

None

Developer impact
----------------

Volume driver maintainers will need report capabilities from their driver to
the scheduler. They can get this information directly from the backend and pass
it right up to the scheduler if it already follows the format specified
earlier. If not, it's up to the driver to parse the response from the backend
in a format the scheduler will understand. If capabilities are not being
reported, the default **False** on features will be done.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thingee

Work Items
----------

* Standardize on the Capabilities here. Right here, right now.

Dependencies
============

None.

Testing
=======

None

Documentation Impact
====================

The developer docs for driver maintainers will need to be updated to include
the list of common capabilities the maintainer needs to have their driver push
that they support. By default capabilities are marked as False, as in not being
supported by the driver.

References
==========

[1] - https://review.openstack.org/#/c/183947/

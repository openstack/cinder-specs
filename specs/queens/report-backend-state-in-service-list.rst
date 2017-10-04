..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Report backend state in service list
====================================

https://blueprints.launchpad.net/cinder/+spec/report-backend-state-in-service-list

Storage driver reports state of backend storage device and let admin operator
know it via service list for maintenance purpose.

Problem description
===================

Currently, Cinder couldn't report backend state to service, operators only
know that cinder-volume process is up, but isn't aware of whether the backend
storage device is ok. Users still can create volume and go to fail over and
over again. To make maintenance easier, operator could query storage device
state via service list and fix the problem more quickly. If device state is
*down*, that means volume creation will fail.


Use Cases
=========

In large scale cloud system, there could be many backends existing in the
system. If volume, snapshot or other resources creation goes to failure,
operators or cloud management system could query the service first and get
the backend device state in every service. If device state is *down*, specify
that storage device has got some problems. Give operators/management system
more information to locate bug more quickly.

Proposed change
===============

* Each driver reports the backend state in "get_volume_stats" by adding
  key/value: "backend_state: up/down"[1].
* When calling 'service list', get this information from scheduler for every
  backend.
* Add 'backend_state: up/down' in response body of service list API if context
  is admin.
* Before all drivers support this feature, if the result of get_volume_stats
  doesn't include the backend state, Cinder will set backend_state to 'up' by
  default.


Alternatives
------------

Add cinder manage command to query backend device state from driver directly.


Data model impact
-----------------

None

REST API impact
---------------

Add backend_state: up/down into response body of service list and also need
a microversions for this feature:

    * GET /v3/{project_id}/os-services

     RESP BODY: {"services": [{"host": "host@backend1",
                               ...,
                               "backend_status": "up",
                              },
                              {"host": "host@backend2",
                               ...,
                               "backend_status": "down",
                              }]
                }

Security impact
---------------

None

Notifications impact
--------------------

None.

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

Driver maintainer needs to add backend state when reporting
volume stats.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  wanghao<wanghao749@huawei.com>


Work Items
----------

* Implement code in Cinder API and scheduler.
* Update cinderclient to support this function.
* Add change to API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests need to be created to cover the code change that
mentioned in "Proposed change".


Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the REST
   API changes.

References
==========

[1]http://docs.openstack.org/developer/cinder/devref/drivers.html

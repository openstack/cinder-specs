..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Cinder API WSGI application under Apache/Nginx
==============================================

https://blueprints.launchpad.net/cinder/+spec/non-eventlet-wsgi-app.

Cinder API uses eventelt as a webserver and WSGI application managing. Eventlet
provides own WSGI application to provide web server functionality.


Problem description
===================

* Cinder API is deployed in other way as a common web application. Apache/Nginx
  is generally used web servers for REST API application.

* Cinder API is runned as a separate service. It means that cloud operator need
  to configure some software to monitor that Cinder API is running.

* Apache/Nginx works better under the real heavy load then eventlet.



Use Cases
=========

* Deploy Cinder API with Apache/Nginx like an any other web applicaction.

* Deploy Cinder API with Apache/Nginx to have load balancing and web applicaion
   monitoring out of the box. E.g. Nginx/uWSGI can restart backend service if
   it stopped by any case.


Proposed change
===============

Provide WSGI application based on used web-framework instead of eventlet. Leave
eventlet-based WSGI application as a default option and make it configurable.

Alternatives
------------

Leave as is and use eventlet for REST API web serving. Use something like
haproxy for API requests load balancing and some watchdog to restart Cinder API
service after shutdown.

Data model impact
-----------------

None.

REST API impact
---------------

None.

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

Potential performance impact could be present if we will have a lot of requests
to Cinder API. Performance impact will be tested with Rally.

Other deployer impact
---------------------

Deployers should configure Apache/Nginx/etc and WSGI module to handle requests
to Cinder API. By default, Cinder API will use eventlet and no deployer impact
will be.

No new configuration options for Cinder will be introduced.

New deployment mode should be supported by Chef cookbooks and Puppet manifests.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Ivan Kolodyazhny <e0ne>

Other contributors:
  Anton Arefiev <aarefiev>

Work Items
----------

* Implement WSGI application based on webob framework.

* Test performance impact with Rally.

* Write documentation how to run Cinder API with Apache/Nginx.

* Implement configuration option in Devstack to support new deployment mode.

* Make sure usage of eventlet doesn't break WSGI in Nginx/Apache.

* Start cross-project initiative to implement this in oslo.


Dependencies
============

None


Testing
=======

Functional tests for new deployment mode will be implemented. We need to test
this feature on every commit on infra with CI.


Documentation Impact
====================

Administrators Guide will be updated.


References
==========


* https://review.openstack.org/#/c/154642/

* https://review.openstack.org/#/c/164035/

* https://review.openstack.org/#/c/196088/

* http://lists.openstack.org/pipermail/openstack-dev/2015-February/057359.html

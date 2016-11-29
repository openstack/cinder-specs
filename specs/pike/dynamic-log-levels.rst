..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Dynamic Log Level control via REST API
======================================

https://blueprints.launchpad.net/cinder/+spec/dynamic-log-levels

Add REST API to control Cinder services' log levels dynamically.

Problem description
===================

To change log levels in a service the service's configuration needs to be
changed and the service restarted.  The restart can be done by restarting the
service itself or by requesting an internal restart via ``SIGHUP`` signal.

In some services a restart is not a big deal, API and scheduler, because they
only operate in the control plane and they don't perform long running
operations, but in other services, Volume and Backup, this is a bigger deal,
because they are in the data plane as well and restarting of a service may take
a long time.

We should be able to change service log levels dynamically as needed, even if
they will revert back to the defaults on restart.

A downside to being able to dynamically change log levels is that we'll no
longer be sure of what log level a service is running at a given time, so we'll
also need a mechanism to query current log levels of a service.

Use Cases
=========

Cloud users are encountering problems when using the cloud and they contact
support, so the system operator starts looking at the logs only to find out
that correct log levels are insufficient to determine the root cause of the
problem and the log levels need to be changed to ``DEBUG``.

Another use case that would be satisfied by the implementation of this spec as
a side product would be when a system administrator wants to confirm Message
Broker connectivity in a service, as the log level query mechanism can be used
as a ping to the service via the Message Broker.

Proposed change
===============

The proposal is to introduce 2 new service REST APIs actions, one to modify
debug levels at runtime and another to query them.  The life of the log level
changes will be the current service run, as they will revert to those defined
in the configuration file upon restart.

Setting the log levels will be possible for all Volume, Scheduler, and Backup
services, but limited in the API service to only the service process that
receives the request since there is no mechanism in place right now to
propagate the request to other API nodes and adding such mechanism for this
feature would be an unnecessary complexity at this point.

This is a reasonable limitation, since API services can be easily restarted
without impacting the cloud because they are only in the control plane and are
usually deployed in an Active/Active configuration.  And if they are not in an
Active/Active configuration then there's only 1 API service running and not
being able to propagate the API log level change isn't such a big deal.

While some operators may prefer to restart the API services to change the log
levels, there may be others that prefer to directly make the dynamic log level
changes to the all the API nodes skipping the load balancer to avoid restarts,
and some others that will just change one API node dynamically skipping the
load balancer and make the test request to that one API node.

The mechanism to set the log level should be versatile enough that no scripting
is necessary when we want to do multiple changes.  The way to achieve this will
be to allow changing log levels to all addressable services or limit by binary
and/or server.

It'll also be possible to decide which log levels to change in the service, so
we'll be able to not only change the log levels of the cinder service itself,
but also those of its libraries (ie. SQLAlchemy library).

Both mechanism will allow setting/querying multiple services but will only work
on services that are up as per DB heartbeats.

Alternatives
------------

An alternative would be to support Dynamic Reconfiguration after modifying
cinder.conf, but that is a considerably bigger problem that will require more
code changes, and while it'll be more powerful it has also some drawbacks,
since it requires access to the nodes to change the configuration of each of
the services and also trigger the reload of each of them.

The benefit of having an API for the log levels is that you don't have to have
access to the infrastructure as you can request the change through the REST API
and then check the logs in the log monitoring service.

Data model impact
-----------------

None

REST API impact
---------------

* Set log level: This will be implemented as a service action like ``enable``
  and ``disable``, but will use the ``set-log`` identifier.  Effective URL
  ``/v3/{tenant_id}/os-services/set-log`` will take following parameters in the
  body:

  * ``binary`` (optional): A string parameter indicating the binary of the
    service to change, it can take following values, ``cinder-volume``,
    ``cinder-scheduler``, ``cinder-backup``, ``cinder-api``, ``*``, ``null``,
    empty string or be missing.  The last four possibilities being equivalent
    to all services.

  * ``server`` (optional): A string parameter indicating the server to change,
    Can be a host or cluster reference - ``host@backend`` or
    ``cluster@backend`` -, or ``null``, empty string, or be missing for all
    servers matching the ``binary``.

  * ``prefix`` (optional): A string indicating the prefix for the log path, for
    example ``cinder.`` or ``sqlalchemy.engine``.  When not present all logs
    will be changed.

  * ``level`` (required): A string with the log level to set, case insensitive,
    accepted values are ``INFO``, ``WARNING``, ``ERROR``, ``DEBUG``.

* Get log level: Service action with ``get-log`` identifier.  Effective URL
  ``/v3/{tenant_id}/os-services/get-log`` will accept the following parameters
  in the body:

  * ``binary`` (optional): A string parameter indicating the binary of the
    service to query, it can take following values, ``*``, empty string,
    ``null``, ``cinder-volume``, ``cinder-scheduler``, ``cinder-backup``, and
    ``cinder-api``.  If missing or ``*`` or the an empty string is passed then
    all binaries will be used.

  * ``server`` (optional): A string parameter indicating the server to query,
    Can be a host or a cluster reference - ``host@backend`` or
    ``cluster@backend``.

  * ``prefix`` (optional): A string indicating the prefix for the log path we
    are querying, for example ``cinder.`` or ``sqlalchemy.engine``.  When not
    present  or the empty string is passed all log levels will be retrieved.

Example response to ``get-log``:

.. code::


   {
      "log_levels":[
          {
             "binary": "cinder-api",
             "host": "hostname1",
             "levels":{
                "cinder.api": "DEBUG",
                "cinder.api.common": "DEBUG"
                "cinder.db.sqlalchemy.api": "DEBUG"
          },
          {
             "binary": "cinder-scheduler",
             "host": "hostname1",
             "levels":{
                "cinder": "DEBUG",
                "cinder.scheduler.manager": "DEBUG"
                "eventlet": "ERROR"
             }
          },
          {
             "binary": "cinder-volume",
             "host": "hostname2@backend#pool",
             "levels":{
                "cinder": "DEBUG",
                "cinder.volume.drivers.rbd": "DEBUG",
                "sqlalchemy": "WARNING"
             }
          }
      ]
   }


Security impact
---------------

None, since it will be using the service update Access Control policy used for
operations like enable, disable, and freeze...

Notifications impact
--------------------

For audit purposes a new notification will be emitted with every dynamic log
level change.

Other end user impact
---------------------

None

Performance Impact
------------------

None besides the possible increase in log quantity when changed to a greater
log level, for example debug.

Other deployer impact
---------------------

None.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gorka Eguileor (geguileo)

Work Items
----------

- Add the set API endpoint and mechanism on the services
- Cinder client support for set action
- Add the get API endpoint and mechanism on the services
- Cinder client support for get action


Dependencies
============

None

Testing
=======

Unittests for new API behavior.

Documentation Impact
====================

Only the changes to the API need to be documented.

References
==========

* `Ocata Design Summit Contributor Meetup Etherpad`__
* `Dynamic Reconfiguration`_

.. _design_meetup: https://etherpad.openstack.org/p/ocata-cinder-summit-meetup
__ _design_meetup

.. _`Dynamic Reconfiguration`:
   https://blueprints.launchpad.net/cinder/+spec/dynamic-reconfiguration/

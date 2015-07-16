..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Use Tooz to replace local file locks
====================================

https://blueprints.launchpad.net/cinder/+spec/cinder-volume-active-active-support

Right now cinder-volume service can run only in Active/Passive HA fashion.
Some of the resons are:

* Local file locks in manager that are protecting resources. For
  example when creating a volume A from volume B a lock on B is created to
  protect B from getting deleted during creation of A.
* Local file locks in drivers like RemoteFS (and others). These locks are used
  to block operations that cannot run concurrently (like taking a snapshot in
  case of RemoteFS-based drivers).

This blueprint proposes a switch to use Tooz_ library for distributed locking.

Problem description
===================

Currently there can be only one active cinder-volume service per volume
backend. This means that to achieve high availability of it we need an external
service (like Pacemaker) that's monitoring state of the c-vol and makes a new
instance active in case of a failure of a previous one. Also this makes c-vol
completely unscalable, because operator cannot just spin up additional services
if load is high and needs to rely on having just one instance.

Local file locks that are used in c-vol's manager and some drivers are
preventing that, because the lock won't be shared between different c-vol
services running on different hosts. This may cause multiple issues, resulting
even in data loss.

Use Cases
=========

Operator would want to deploy multiple cinder-volume services. The motivation
may be to run multiple cinder-volume attached to a single storage backend to
increase throughput of requests and therefore performance. Another advantage is
increased resiliency to hardware failure when running multiple instances of the
service.

Cloud user will get a better cloud.

Proposed change
===============

At Mitaka Design Summit there was a `session about allowing projects to have a
hard requirement on DLM software`_. Main conclusions from the session are:

* Projects can hard-depend on having a DLM.
* Tooz_ will be the abstraction layer.

That's why proposed solution is to convert current locks that are local to use
Tooz_ abstraction layer.

This would require unification of current approaches (as some locks are done
through cinder.utils.synchronized method and some are using
oslo.concurrency.lockutils directly). By default Tooz would be configured to
use file locks, so everything will work as today. If operator would want to run
multiple cinder-volume services he would need to configure Tooz backend service
and set it in cinder.conf. Currently most reliable Tooz backends are ZooKeeper
and Redis.

Redis backend in Tooz requires sending periodical heartbeats, so cinder-volume
manager will start a new thread that will take care of that. This will be a
regular thread (non-eventlet) to be safe from situations when eventlet
greenthreads are blocked. It is intended to spawn this thread only when Tooz
needs it, so this won't be done if FileDriver or ZooKeeper is used as backend.

Alternatives
------------

To solve the problem with locks in RemoteFS-based volume drivers we may simply
deprecate them and make operators switch to use lock-free drivers. This may be
very hard to achieve because according to `OpenStack Users Survey`_ 19% of
Cinder deployments use these drivers. Also some non-RemoteFS-based drivers are
using local locks too.

We could also replace current locks with some DB-based locking. This was
proposed by gegulieo in specs to remove local locks from the maanger_ and from
drivers_, but increased the complexity of the solution and potentially required
more testing than relying on a broadly used DLM software.

In the future we will probably want to try to remove locks from volume manager
using other means (for example state-based locking). This will make it possible
to run c-vol in A/A manner without DLM (when running a volume driver that
doesn't do locking).

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

None

Performance Impact
------------------

Using an external service for locking will affect the performance. There are
some `performance tests done by geguileo`_. Also having a heartbeat thread may
add a tiny performance overhead.

These will be non-existent if user would stick to file locks and decide to run
c-vol exactly as it works today.

Other deployer impact
---------------------

Deployer will have a new options in a section [coordination]:

* `backend_url=file://$state_path` - Tooz backend connection string.
* `heartbeat=1.0` - number of seconds between heartbeats for distributed
  coordination.
* `initial_reconnect_backoff=0.1` - number of seconds to wait after failed
  reconnection to Tooz backend.
* `max_reconnect_backoff=60.0` - Maximum number of seconds between sequential
  reconnection retries to Tooz backend.

Deployment tools maintainers will need to decide if they want to use new
possibility. If so - they will need to setup a Tooz backend (ZooKeeper, Redis,
or less recommended memcached) in their environment.

Developer impact
----------------

Developer would need to use locks in cinder-volume service only through Tooz
library abstraction layer and be aware that there can run multiple instances of
the service.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Michal Dulko (dulek)

Other contributors:
  Szymon Wroblewski (bluex-pl)

Work Items
----------

* Tooz locking implementation in Cinder (done).
* Switch current locks to use Tooz implementation.

  * We have some `work already done`_. We should split it for each driver and
    work with driver maintainers to get patches merged.

* Add DevStack patches to set up a CI testing Cinder with Redis or ZooKeeper as
  Tooz backend.

Dependencies
============

None

Testing
=======

Unit tests for Tooz code will be added and a CI configured to test Cinder
with Redis or ZooKeeper as lock backend will be set up. Possibly we can do that
with multinode Tempest.

Documentation Impact
====================

Documentation and Openstack HA Guide will need to be updated to include
instructions on how to configure Tooz and deploy cinder-volume in A/A.

References
==========

.. _Tooz: http://docs.openstack.org/developer/tooz/
.. _Openstack Users Survey: http://www.openstack.org/assets/survey/Public-User-Survey-Report.pdf
.. _performance tests done by geguileo: https://github.com/Akrog/test-cinder-atomic-states
.. _session about allowing projects to have a hard requirement on DLM software: https://etherpad.openstack.org/p/mitaka-cross-project-dlm
.. _manager: https://review.openstack.org/#/c/237602/
.. _drivers: https://review.openstack.org/#/c/237604/
.. _work already done: https://review.openstack.org/#/c/185646/

* https://etherpad.openstack.org/p/cinder-active-active-vol-service-issues
* http://lists.openstack.org/pipermail/openstack-dev/2015-June/068151.html
* https://review.openstack.org/#/c/183537/
* https://www.youtube.com/watch?v=Fs9LC_sjnRM

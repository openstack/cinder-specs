..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
New Core APIs for Discovering System Capabilities
=================================================

https://blueprints.launchpad.net/cinder/+spec/discovering-system-capabilities

There needs to be a way for Horizon to discover the capabilities of the remote
block storage deployment so that it can enable/disable the UI widgets
that exercise those capabilities. Consider the volume backup functionality
which may or may not be supported by a cinder deployment. If Horizon could
programmatically detect the presence of this capability, it can enable the
volume backup actions in the UI. This will obviate the need for the operator
to manually set appropriate fields in Horizon's config to signal the
availability of this feature, which is the case today. We propose adding
a set of APIs that would return the "capabilities" available to the current
user. These can be used by Horizon and other clients to configure themselves
without the need for config files that are manually kept in sync.

This blueprint grew out of the design discussion in [1]_ and [2]_ for the bug
#1334856 [3]_ filed by jgriffith.

Problem description
===================

Horizon allows users to create volume backups via the UI, if the underlying
cinder deployment supports the same. Not all cinder deployments support this
functionality, so Horizon needs a way to know when it is supported so that
it can show the UI elements for backup.

Today, Horizon uses [5]_ a config setting "enable_backup" for controlling the
enable status of volume backup operations in the UI. This needs to be manually
set/unset by the operator based on the status of the cinder-backup service.
This is not only cumbersome but also prone to operator error.

The volume backup API is implemented on the server side by the cinder-backup
service. This service is absent when the cinder deployment does not support
volume backups. This is the case, e.g., for devstack with default config.

The "os-services" API extension shows the state of all backend services.
This suggests a way by which Horizon could get to know the existence of the
cinder-backup service. However this has two problems. The first is that the
os-services API is admin-only whereas the backup operation is available to
even non-admin users. The second and more important problem is that not all
features have corresponding services or extension APIs that could be used to
compute the support for that feature in a similar way.

This problem raises its head in non UI domains also. Consider the OpenStack
Heat project. Heat templates allows the option of backing up a volume upon the
deletion of the corresponding volume resource in the template. However,
as there is no way to know whether backup capability is supported by a
Cinder deployment, this can result in late failures during a volume delete.
This ultimately prevents interoperability of orchestration templates across
clouds.

It is easy to see that the above problem is not limited to just the backup
feature but is much more general. Horizon needs a way to programmatically
know which capabilities are supported by the remote block storage service so
that it can enable only those UI widgets that deal with these capabilities
and disable others.

Use Cases
=========

The solution proposed in this spec allows building a dynamic, self-configuring
Horizon UI. This unburdens the operator from manually configuring Horizon and
constantly keeping it in sync with the capabilities of the Cinder deployment.

This solution can be immediately applied to solve the volume backup enable
status problem in Horizon. No longer will the operator need to set the
"enable_backup" in Horizon config; Horizon will programmatically get the
status using the proposed API.

Other capability which can similarly gain is the replication capability, which
is only supported by certain volume types (actually "volume backends", but
normally volume backends are more or less tied to volume types). Note that
feature will be listed by a different capability API than the one that lists
the backup feature. See details below.

The proposed set of APIs can also be used in custom UIs or user scripts for
the same reasons that we envision Horizon using it for.

Proposed change
===============

A new, public Block Storage microversion API that returns the set of all
capabilities for a particular user and resource. Note that the service (or
the deployment) itself is considered as a resource (the root resource).

The capabilities of a system can be defined at multiple levels. The
top-level capabilities such as backup capability are defined at the service
level (or the root resource level). So, the question "Does this cinder service
instance have backup capability?" makes sense by itself. Lower level
capabilities are defined on specific (REST) resources. For example,
replication feature is defined on a volume type. So, it doesn't make sense to
ask "Does this cinder service instance have the replication capability?";
instead, we need to ask "Does the volume type xyz have the replication
capability?".

While we can define a capabilities API for every resource, in practice it will
be required only a small number of resources. For the Cinder API, it is only
required today for the service and volume type resources.

It is not possible to avoid multiple levels of capability APIs. The
lower level capability APIs require input (the resource type) which is not
available at the top level.

The different levels of capability APIs also align with the natural
organization of any UI (e.g. Horizon). On the home page, the UI only shows
the widgets corresponding to top level capabilities. It is possible to
navigate to deeper levels by selecting widgets on the home page. For
example, selecting a particular volume type can lead a page showing widgets
that correspond to the capabilities of that particular volume type. By making
the calls to appropriate level capabilities API, Horizon can programmatically
configure itself at every level.

Note that a cinder instance may implement a particular feature but may not
allow a particular user to access the same. So, the capabilities API
should not only consider whether the service implements the particular feature
but also if the current user is privileged to access the same. The "privilege"
information is already available in the policy.json file that maps different
operations to the users that can access them. So, any capapbility API
implementation must make use of this file.

The detection of whether a particular deployment/resource implements a
particular capability varies by the capability itself. For the backup
capability, it could be the presence of the cinder-backup service. For
replication feature, it could be statically set when the volume type is
created or it could be fetched from the driver somehow.

It is not possible to implement the capabilities API based solely on the info
in the policy.json file as the policy file does not allow defining rules per
resource. For example, we can allow replication operation for all users but we
cannot constrain it to a specific volume type (which is deal breaker since not
all volume types support replication).

It is easy to see that all capabilities APIs should be public, i.e. accessible
by any user.

The key contribution of this BP is identifying and proposing an API pattern,
namely "one capability API per resource" (of course if some resource doesn't
need it, we don't need to define a capability API for it). This idea is simple
but powerful and also reusable across all OpenStack projects.

While this BP proposes an API pattern, we will only implement the top level
capabilities API. This will return only "volume-backup" for now but can be
augmented as new capabilities are added to Cinder at the service level. For
now, the detection of the backup capability would be similar to the
os-services extension. It will check if the cinder-backup service is enabled
or not by checking the services table from cinder database and if so return
"volume-backup" in the response.

A brief note on the presence of capabilities like APIs in Cinder today: There
is a "volume capabilities" extension API [6]_, but it is defined at the
service level (``/v2/{tenant_id}/capabilities/{hostname}``) rather than at
the volume type level. The "show volume type details" API [8]_ stuffs the
capabilities in a catch-all "extra_specs" property. If this BP is approved,
we will need to rationalize these existing APIs with the new capabilities
APIs that will be defined. This is not in scope for this sepc.

Alternatives
------------

* Make the existing os-services extension public: This will expose the private
  cloud internals to the tenant. This is a security hole and hence makes this
  alternative infeasible. Also, there may not be a 1:1 correspondence between
  capabilities and services.

* Split the existing os-services extension API into public and private halves,
  with the public part exposing limited information.

  We could modify the services:index action to take a details=true/false
  parameter: http://{cinder-endpoint}/v2/{tenant-id}/os-services?details=false

  And define different policies for detail=true (admin_api) and detail=false
  ("" i.e. unrestricted).

  * "volume_extension:services:index_with_details": "rule:admin_api"

  * "volume_extension:services:index_without_details": ""

  It is not clear if this can be implemented in a backward compatible way and
  also whether there is precedence for splitting the policy of a single API
  call based on parameters. Also, as mentioned above, there may not be a 1:1
  correspondence between capabilities and services.

* Re-use the existing "list extensions" public API [7]_. This was proposed by
  dulek in [2]_. First, there may not be a 1:1 correspondence between
  capabilities and extensions (although it is true for the volume backup
  case). Even if it were always true, the operator would need to prune
  cinder.conf (manually!) so that it lists only those extensions that are
  actually supported. As explained in [2]_, there is no easy way to do that.
  Also, as noted by duncant, this breaks the existing semantics - many
  deployments have the API extensions enabled (as it comes by default) without
  the service being actually running. So the check return value would mean
  different things on different systems.

Data model impact
-----------------

None. As explained, we use the existing services table for the volume backup
capability detection. Future capability additions may use different resources
and algorithms.

REST API impact
---------------

We give instances of the "capabilities API pattern" for three resources,
including the service itself.


* ``GET /v3.x/{tenant id}/capabilities``

  Returns the set of capabilities of the underlying block storage deployment
  at the service level.

  Normal http response code(s): 200

  Response is a list of capabilities. Each capability is a simple noun or
  hyphenated compound noun. E.g:

.. code-block:: rest

  {
    "capabilities": [
      {
        "name": "volume-backup",
        "description": "Allows creating backups of volumes."
      },
      {
        "name": "other-capability",
        "description": "Other capability description."
      },
    ]
  }

* ``GET /v3.x/{tenant_id}/types/{volume_type_id}/capabilities``

  Returns the set of capabilities of a particular volume type.

  Normal http response code(s): 200

  Response is a list of capabilities. E.g:

.. code-block:: rest

  {
    "capabilities": [
      {
        "name": "replication",
        "description": "Allows replication of volumes."
      },
      {
        "name": "other-capability",
        "description": "Other capability description."
      },
    ]
  }

* General example:

     .. code-block:: json

        GET /v3.x/{tenant_id}/<some-resource-collection>/
        {some-resource-instance}/capabilities

  Returns the set of capabilities of some-resource-instance.

Security impact
---------------

None. Exposing an abstract set of system capabilities should be safe. These
capabilities can be gleaned from the available actions in the UI in any case
(e.g. backup UI widget is visible implies volume backup capability exists).

Notifications impact
--------------------

None

Other end user impact
---------------------

This change is transparent to the user although the user can use this API in
a similar way as Horizon for custom UIs or management scripts.

Performance Impact
------------------

These are new APIs and should not affect any existing APIs
or code paths.

Other deployer impact
---------------------

Operator no longer has to manually set "enable_backup" in Horizon config
settings file. Back-compat story for this Horizon config change is out of
scope for this spec.

Developer impact
----------------

The developer will need to be aware of the capabilities API pattern and
evaluate if any new optional functionality he/she plans to add to a Cinder
service or a lower level resource (e.g. volume type) may benefit from
being exposed via this API. The developer may first need to define a
capability API for that resource.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dramakri

Work Items
----------

* Implement the proposed public microversion capabilities API at the
  service level
* Implement at least the backup capability detection
* Add test cases
* Update API docs


Dependencies
============

None


Testing
=======

Unit and functional test cases need to be added to validate this new API.


Documentation Impact
====================

* New API and client call in Cinder needs to be documented.
* Changes to Horizon config setting needs to be documented.


References
==========

.. [1] http://lists.openstack.org/pipermail/openstack-dev/2015-October/077209.html
.. [2] http://lists.openstack.org/pipermail/openstack-dev/2016-April/092365.html
.. [3] https://bugs.launchpad.net/cinder/+bug/1334856
.. [4] https://github.com/openstack/cinder/blob/master/etc/cinder/policy.json
.. [5] http://docs.openstack.org/developer/horizon/topics/settings.html
.. [6] http://developer.openstack.org/api-ref-blockstorage-v2.html#capabilities-v2
.. [7] http://developer.openstack.org/api-ref-blockstorage-v2.html#volumes-v2-extensions
.. [8] http://developer.openstack.org/api-ref-blockstorage-v2.html#showVolumeType

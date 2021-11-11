..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Remove the need for project_id from API endpoints
=================================================

https://blueprints.launchpad.net/cinder/+spec/project-id-optional-in-urls

Problem description
===================

Cinder currently assumes API URLs include an associated project_id. Including
a project_id in the URL is redundant because the project_id is typically
available in the keystone context for the API request. In addition, requiring
a project_id is incompatible with keystone's notion of system scope, in which
an API is not associated with a specific project. System scoped API requests
will not have a project_id, and therefore the URL for those requests should
not include a project_id.

Furthermore, cinder's inclusion of a project_id in its API URLs is out of sync
with other OpenStack services. The Images (glance API v2), Compute (nova API
as of v2.16), Identity (keystone API v3) and Shared File Systems (manila API
as of v2.60) services do not require a project_id in their API URLs.

Use Cases
=========

A cloud administrator wishes to utilize keystone personas in order to grant
appropriate access permissions to users that are assigned to different roles,
e.g. lab operators and project administrators. Lab personnel are typically
required to manage system-wide cinder resources, such a storage backends and
volume types. However, these personnel should not have access to project
specific resources, such as the cinder volumes. Likewise, project
administrators need the ability to manage the resources owned by projects for
which they have been given responsibility, but should have limited access to
system wide resources, such as the ability to configure volume types.

In order to support system scoped personas, the URL for API requests should
not be required to include a project_id.

Proposed change
===============

This spec proposes the cinder API be enhanced to not require a project_id in
the API's URL. Inclusion of a project_id in the URL would be optional, which
means the proposed change retains backward compatibility. A new microversion
will be introduced so that clients may know whether inclusion of a project_id
in API URLs is optional or mandatory.

The microversion will only be used as an indicator that the API can handle
URLs without project IDs, and this applies to all requests reqardless of the
microversion in the request.  For example, if the new proposed microversion is
v3.67, an API node serving v3.67 or greater will properly route requests
without project_id even if you ask for v3.0.  Likewise, it will properly route
requests to URLs containing project_id even if you ask for v3.67.

The proposal does not introduce any behavioral or functional changes to the
API. The same API method will be invoked regardless of whether the URL
includes a project_id, and the actual project_id associated with the call will
be the one contained in the keystone context for the API request. The current
behavior of returning HTTP 400 will be retained if the request URL contains a
project_id that doesn't match the one in the keystone context.

Alternatives
------------

A dummy project_id value (perhaps all zeroes) could be defined for use by
system scoped API requests. This would not be an elegant solution, and would
require changes in keystone.

Cloud administrators could be required to create an actual project solely for
use when making system scoped API requests. This is also not elegant, a shifts
the burden of managing system scope onto cloud administrators.

Data model impact
-----------------

None

REST API impact
---------------

The behavior of each API method will be unchanged. However, the API routing
will be enhanced to provide duplicate routes to each method.

#. The existing route for which a project_id is in the URL
#. A new route for when the URL does not include a project_id


Security impact
---------------

There is no security impact because keystone will continue to provide an
authorization context for every API request, regardless of whether the request
URL includes a project_id.

Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

End users typically interact with the cinder API through other clients, such
as the python-cinderclient, python-openstackclient, or openstacksdk. In that
regard, there is no direct impact on the end user.

End users who interact with the API directly using tools such as ``curl`` may
continue to include a project_id in the URL. They will have the option of not
including a project_id in the URL.

Performance Impact
------------------

None

Other deployer impact
---------------------

Deployment tools that create cinder endpoints in the keystone catalog no
longer need to use project_id templating. Endpoints in the catalog should look
like this:

.. code-block:: console

  $ openstack endpoint list --service cinderv3 -c 'Service Name' -c URL
  +--------------+---------------------------------+
  | Service Name | URL                             |
  +--------------+---------------------------------+
  | cinderv3     | http://192.168.100.30/volume/v3 |
  +--------------+---------------------------------+

rather than this:

.. code-block:: console

  $ openstack endpoint list --service cinderv3 -c 'Service Name' -c URL
  +--------------+------------------------------------------------+
  | Service Name | URL                                            |
  +--------------+------------------------------------------------+
  | cinderv3     | http://192.168.100.30/volume/v3/$(project_id)s |
  +--------------+------------------------------------------------+

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee: abishop

Work Items
----------

- Define a new microversion
- Add API routes to connect API requests with no project_id in the URL to the
  associated method
- Add or enhance unit tests
- Update devstack so the cinder endpoint doesn't use project_id templating

Dependencies
============

None

Testing
=======

Existing tempest tests will certainly verify whether the API functions
correctly. The challenge is verifying both the legacy behavior, where URLs
include a project_id, and the new behavior when URLs don't include a
project_id. One possiblity is to employ a periodic zuul job that specifically
tests the legacy behavior. The details can be worked out during the
development phase.

Documentation Impact
====================

The existing API reference is largely unaffected. This is because the
reference guide is unbranched, and older versions of cinder require a
project_id. A brief note will be added to explain the project_id in the API
reference is optional as of the new microversion.

References
==========

- OpenStack's `Support Common Operator & End User Personas
  <https://review.opendev.org/c/openstack/governance/+/815158>`_ community goal
- Cinder's `Secure RBAC Ready
  <https://review.opendev.org/c/openstack/cinder-specs/+/809741>`_ spec

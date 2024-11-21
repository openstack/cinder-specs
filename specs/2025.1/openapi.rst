..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
OpenAPI Schemas
===============

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/openapi

We would like to start documenting our APIs in an industry-standard,
machine-readable manner. Doing so opens up many opportunities for both
OpenStack developer and OpenStack users alike, notably the ability to both
auto-generate and auto-validate both client tooling and documentation alike.
Of the many API description languages available, OpenAPI (fka "Swagger")
appears to be the one with both the largest developer mind share and the one
that would be the best fit for OpenStack due to the existing tooling used in
many OpenStack services, thus we would opt to use this format.

Problem description
===================

The history of API description languages has been mainly a history of
half-baked ideas, unnecessary complication, and in general lots of failure.
This history has been reflected in OpenStack's own history of attempting to
document APIs, starting with our early use of WADL through to our experiments
with Swagger 2.0 and RAML, leading to today's use of our custom ``os_api_ref``
project, built on reStructuredText and Sphinx.

It is only in recent years that things have started to stabilise somewhat, with
the development of widely used API description languages like OpenAPI, RAML and
API Blueprint, as well as supporting SaaS tools such as Postman and Apigee.
OpenAPI in particular has seen broad adoption across multiple sectors, with
sites as varied as `CloudFlare`__ and `GitHub`__ providing OpenAPI schemas for
their APIs. OpenAPI has evolved significantly in recent years and now supports
a wide variety of API patterns including things like webhooks. Even more
beneficial for OpenStack, OpenAPI 3.1 is a full superset of JSON Schema meaning
we have the ability to re-use much of the validation we already have.

.. __: https://blog.cloudflare.com/open-api-transition
.. __: https://github.com/github/rest-api-description

Use Cases
=========

As an end user, I would like to have access to machine-readable, fully
validated documentation for the APIs I will be interacting with.

As an end user, I want statically viewable documentation hosted as part of the
existing docs site without requiring a running instance of Cinder.

As an SDK/client developer, I would like to be able to auto-generate bindings
and clients, promoting consistency and minimising the amount of manual work
needed to develop and maintain these.

As a Cinder developer, I would like to have a verified API specification that I
can use should I need to replace the web framework/libraries we use in the
event they are no longer maintained.

Proposed change
===============

This effort can be broken into a number of distinct steps:

- Add missing request body and query string schemas

  These schemas will merely validate what is already allowed, which means
  extensive use of ``"additionalProperties": true`` or empty schemas.

  Once these are added, tests will be added to ensure all API resource have
  appropriate schemas.

- Add response body schemas

  These will be sourced from both existing OpenAPI schemas, currently published
  at `opendev.org/openstack/codegenerator`__, and from new schemas
  auto-generated from JSON response bodies generated in tests and manually
  modified handle things like enum values.

  Once these are added, tests will be added to ensure all API resource have
  appropriate response body schemas. In addition, we will add a new
  configuration option that will control how we do verification at
  the API layer, ``[api] response_validation``. This will be an enum value with
  three options:

  ``error``
    Raise a HTTP 500 (Server Error) in the event that an API returns an
    "invalid" response.

    This will be the default in CI i.e. for our unit, functional and
    integration tests. Eventually this could be the default for production
    also (but probably not).

  ``warn``
    Log a warning about an "invalid" response, prompting operations to file a
    bug report against Cinder.

    This will be initial (and likely forever) default in production.

  ``ignore``
    Disable API response body validation entirely. This is an escape hatch in
    case we mess up.

.. __: https://opendev.org/openstack/codegenerator

.. note::

    The development of tooling required to gather these JSON Schema schemas and
    generate an OpenAPI schema will not be developed inside Cinder and is
    therefore not covered by this spec. Cinder will merely consume the
    resulting tooling for use in documentation.

Alternatives
------------

- Use a different tool

  OpenAPI has been chosen because it is the most widely used API description
  language available and matches well with our existing use of JSON Schema for
  API validation.

- Maintain these specs out-of-tree

  This prevents us testing these on each commit to Cinder and means work that
  could be spread across multiple teams is instead focused on one small team.

Data model impact
-----------------

None.

REST API impact
---------------

There will be no direct REST API impact. Users will see HTTP 500 error if they
set ``[api] response_validation = error`` and encounter an invalid response,
however, we will not encourage use of this option in production and will
instead focus on validating this ourselves in CI.

We may wish to address issues that are uncovered as we add schemas, but this
work is considered secondary to this effort and can be tackled separately.

Security impact
---------------

None.

Active/Active HA impact
-----------------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

This should be very beneficial for users who are interested in developing
client and bindings for OpenStack. In particular, this should (after an initial
effort in code generation) reduce the workload of the SDK team as well as teams
outside of OpenStack that work on client tooling such as the Gophercloud team.

Performance Impact
------------------

There will be a minimal impact on API performance when validation is enabled as
we will now verify both requests and responses for all API resources. Given our
existing extensive use of JSON Schema for API validation, it is expected that
this should not be a significant issue. In addition, we will not recommend
enabling this option in production.

Other deployer impact
---------------------

As noted previously, there will be one new config option, ``[api]
response_validation``. Operators may see increased warnings in their logs due
to incomplete schemas, but most if not all of these issues should be ironed out
by our CI coverage.

Developer impact
----------------

Developers working on the API microversions will now be encouraged to provide
JSON Schema schemas for both requests and responses.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stephenfinucane

Other contributors:
  gtema

Work Items
----------

- Add missing request body schemas
- Add tests to validate existence of request body schemas
- Add missing query string schemas
- Add tests to validate existence of query string schemas
- Add response body schemas
- Add decorator to validate response body schemas against response
- Add tests to validate existence of response body schemas

Dependencies
============

The actual generation of an OpenAPI documentation will be achieved via a
separate tool. It is not yet determined if this tool will live inside an
existing project, such as ``os_api_ref`` or ``openstacksdk``, or inside a
wholly new project. In any case, it is envisaged that this tool will handle
OpenStack-specific nuances like microversions that don't map 1:1 to OpenAPI
concepts in a consistent and documented fashion.

Testing
=======

Unit tests will ensure that schemas eventually exist for request bodies, query
strings, and response bodies.

Unit, functional and integration tests will all work together to ensure that
response body schemas match real responses by setting ``[api]
response_validation`` to ``error``.

Documentation Impact
====================

Initially there should be no impact as we will continue to use ``os_api_ref``
as-is for our ``api-ref`` docs. Eventually we will replace or extend this
extension to generate documentation from our OpenAPI schema.

References
==========

None.

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2024.02
     - Introduced
   * - 2025.01
     - Re-proposed

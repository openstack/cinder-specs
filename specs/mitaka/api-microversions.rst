..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Cinder API Microversions
==========================================

We need a way to be able to introduce changes to the REST API to both
fix bugs and add new features. Some of these changes are backwards
incompatible and we currently have no way of doing this. This becomes
especially important to introduce backwards incompatible changes to the
Nova - Cinder API.
Credit and thanks to the authors of the Nova spec upon which this is based,
especially Christopeher Yeoh and Sean Dague, and to the Manila team,
especially Clinton Knight.

Blueprint:
https://blueprints.launchpad.net/cinder/+spec/cinder-api-microversions

Problem description
===================

Many changes to the Cinder REST API require changes to the consumers of the
API. For example, If we need to add a required parameter to a method that is
called by Nova, we'd need both the Nova calling code and the cinderclient that
Nova uses to change. But newer Cinder versions with the change must work with
older Nova versions, and there is no mechanism for this at the moment. Adding
microversions will solve this problem.
With microversions, the highest supported version will be negotiated by a field
in the HTTP header that is sent to the Cinder API. In the case where the field
'versions' is not sent (i.e. clients and scripts that pre-date this change),
then the lowest supported version would be used. This means that our current
Cinder API v2 would be the default, and consumers of the API that wished to
use a newer version could do so.

Not in Scope: Experimental APIs. This will be done separately.
Note that Experimental APIs are only different in how we choose to
use the API microversions. For Nova, a microversion does not have to
be backwards compatible, and a microversion might remove an API. In this
definition of microversions, the Experimental APIs that are used by
Manila are no different from microversions that are used by Nova.
This spec assumes that Cinder will take the route used by Manila, and
use the Experimental API HTTP header to indicate an API that might
cause a backwards incompatible change and/or be removed.
These policies do not have to be determined now, and can be decided
at the time that patches with new microversions are merged.

Use Cases
=========

* Allows developers to modify the Cinder API in backwards compatible
  way and signal to users of the API dynamically that the change is
  available without having to create a new API extension.

* Allows developers to modify the Cinder API in a non backwards
  compatible way whilst still supporting the old behaviour. Users of
  the REST API are able to decide if they want the Cinder API to behave
  in the new or old manner on a per request basis. Deployers are able
  to make new backwards incompatible features available without
  removing support for prior behaviour as long as there is support
  to do this by developers.

* Users of the REST API are able to, on a per request basis, decide
  which version of the API they want to use (assuming the deployer
  supports the version they want). This means that a deployer who does
  not upgrade will not break compatibility, and clients that do not upgrade
  will also remain compatible.

Proposed change
===============
Cinder will use a framework we will call 'API Microversions' for allowing
changes to the API while preserving backward compatibility. The basic idea is
that a user has to explicitly ask for their request to be treated with a
particular version of the API. So breaking changes can be added to the API
without breaking users who don't specifically ask for it. This is done with
an HTTP header X-OpenStack-Cinder-API-Version which is a monotonically
increasing semantic version number starting from 2.1.

If a user makes a request without specifying a version, they will get the
DEFAULT_API_VERSION as defined in cinder/api/openstack/wsgi.py. This value is
currently 2.0 and is expected to remain so for quite a long time.

For the purposes of this discussion, "the API" is all core and
optional extensions in the Cinder tree.
Please note that extensions cannot be versioned. It has been discussed that
the Cinder team should therefore move the API extensions into core. This move
of extensions to core will take some time, and should not block making changes
to the API. Since changes to the API extensions cannot be versioned using the
API-microversions decorator, we will have to accept this until the work to move
extensions to core is completed.

Versioning of the API should be a single monotonic counter. It will be
of the form X.Y where it follows the following convention:

* X will only be changed if a significant backwards incompatible
  API change is made which affects the API as whole. That is, something
  that is only very very rarely incremented.
* Y when you make any change to the API. Note that this includes
  semantic changes which may not affect the input or output formats or
  even originate in the API code layer. We are not distinguishing
  between backwards compatible and backwards incompatible changes in
  the versioning system. It will, however, be made clear in the
  documentation as to what is a backwards compatible change and what
  is a backwards incompatible one.


Note that groups of similar changes across the API will not be made
under a single version bump. This will minimise the impact on users as
they can control changes that they want to be exposed to.

A backwards compatible change is defined as one which would be allowed
under the `OpenStack API Change Guidelines`_

A version response would look as follows for GET http://<cinder_URL>:8776

::

 {
    "versions": [
        {
            "id": "v2.0",
            "links": [
                {
                    "href": "http://docs.openstack.org/",
                    "rel": "describedby",
                    "type": "text/html"
                },
                {
                    "href": "http://10.10.10.77:8776/v2/",
                    "rel": "self"
                }
            ],
            "media-types": [
                {
                    "base": "application/json",
                    "type": "application/vnd.openstack.volume+json;version=1"
                },
                {
                    "base": "application/xml",
                    "type": "application/vnd.openstack.volume+xml;version=1"
                }
            ],
            "min_version": "",
            "status": "SUPPORTED",
            "updated": "2014-06-28T12:20:21Z",
            "version": ""
        },
        {
            "id": "v2.1",
            "links": [
                {
                    "href": "http://docs.openstack.org/",
                    "rel": "describedby",
                    "type": "text/html"
                },
                {
                    "href": "http://10.10.10.77:8776/v2/",
                    "rel": "self"
                }
            ],
            "media-types": [
                {
                    "base": "application/json",
                    "type": "application/vnd.openstack.volume+json;version=1"
                },
                {
                    "base": "application/xml",
                    "type": "application/vnd.openstack.volume+xml;version=1"
                }
            ],
            "min_version": "2.0",
            "status": "CURRENT",
            "updated": "2015-09-16T11:33:21Z",
            "version": "2.1"
        }
    ]
 }

This specifies the min and max version that the server can
understand. min_version will start at 2.0 representing the v2.0 API.
Note that this assumes we will drop support for v1.0 in Mitaka.
It may eventually be increased if there are support burdens we don't feel are
adequate to support.
This response indicates a version of 2.1 as the current
version. This number would change with each monotonic increment of the API
microversion.

Client Interaction
-----------------------

A client specifies the version of the API they want via the following
approach, a new header::

  X-OpenStack-Cinder-API-Version: 2.114

This conceptually acts like the accept header. This is a global API
version.

Semantically this means:

* If X-OpenStack-Cinder-API-Version is not provided, act as if min_version was
  sent.

* If X-OpenStack-Cinder-API-Version is sent, respond with the API at that
  version. If that's outside of the range of versions supported,
  return 406 Not Acceptable.

* If X-OpenStack-Cinder-API-Version: latest (special keyword) return
  max_version of the API.

NOTE about use of "latest" as a microversion:
A client should never use "latest" when calling the Cinder API, since it
is possible that the client does not have support for the latest server API
microversion. The use of "latest" is strictly for testing. An experimental
(non-gating) Tempeset test should use the microversion "latest" to detect
when the Tempest tests themselves must be updated. This experimental test will
fail using "latest" when the Tempest tests are out of date, and we will thus
have an automated way to detect when Tempest must be updated.

This means out of the box, with an old client, an OpenStack
installation will return vanilla OpenStack responses at v2. The user
or SDK will have to ask for something different in order to get new
features.

Two extra headers are always returned in the response:

* X-OpenStack-Cinder-API-Version: version_number
* Vary: X-OpenStack-Cinder-API-Version

The first header specifies the version number of the API which was
executed.

The second header is used as a hint to caching proxies that the
response is also dependent on the X-OpenStack-Cinder-API-Version and
not just the body and query parameters. See RFC 2616 section 14.44 for
details.

Implementation design details
-----------------------------

On each request the X-OpenStack-Cinder-API-Version header string will be
converted to an APIVersionRequest object in the wsgi code. Routing
will occur in the usual manner with the version object attached to the
request object (which all API methods expect). The API methods can
then use this to determine their behaviour to the incoming request.

Types of changes we will need to support::

* Status code changes (success and error codes)
* Allowable body parameters (affects input validation schemas too)
* Allowable url parameters
* General semantic changes
* Data returned in response
* Removal of resources in the API
* Removal of fields in a response object or changing the layout of the response

Note: This list is not meant to be an exhaustive list

Within a controller case, methods can be marked with a decorator
to indicate what API versions they implement. For example::

  @api_version(min_version='2.0', max_version='2.9')
  def show(self, req, id):
     pass

  @api_version(min_version='2.17')
  def show(self, req, id):
     pass

An incoming request for version 2.2 of the API would end up
executing the first method, whilst an incoming request for version
2.17 of the API would result in the second being executed.

For cases where the method implementations are very similar with just
minor differences a lot of duplicated code can be avoided by versioning
internal methods intead. For example::

  @api_version(min_version='2.0')
  def _version_specific_func(self, req, arg1):
     pass

  @api_version(min_version='2.5')
  def _version_specific_func(self, req, arg1):
     pass

  def show(self, req, id):
     .... common stuff ....
     self._version_specific_func(req, "foo")
        .... common stuff ....

Reducing the duplicated code minimizes maintenance
overhead. So the technique we use would depend on individual
circumstances of what code is common/different and where in the method
it is.

A version object is passed down to the method attached to the request
object so it is also possible to do very specific checks in a
method. For example::

  def show(self, req, id):
    .... stuff ....

    if req.ver_obj.matches(start_version, end_version):
      .... Do version specific stuff ....

    ....  stuff ....

Note that end_version is optional in which case it will match any
version greater than or equal to start_version.

Some prototype code which explains how this work is available here:
https://github.com/scottdangelo/TestCinderAPImicroversions

The validation schema decorator would also need to be extended to support
versioning

@validation.schema(schema_definition, min_version, max_version)

Note that both min_version and max_version would be optional
parameters.

A method, extension, or a field in a request or response can be
removed from the API by specifying a max_version::

  @api_version(min_version='2.0', max_version='2.9')
  def show(self, req, id):
    ....  stuff ....

If a request for version 2.11 is made by a client, the client will
receive a 404 as if the method does not exist at all. If the minimum
version of the API as whole was brought up to 2.10 then the extension
itself could then be removed.

The minimum version of the API as a whole would only be increased by a
consensus decision between Cinder developers who have the overhead of
maintaining backwards compatibility and deployers and users who want
backwards compatibility forever.

Because we have a monotonically increasing version number across the
whole of the API rather than versioning individual plugins we will have
potential merge conflicts like we currently have with DB migration
changesets. Sorry, I don't believe there is any way around this, but
welcome any suggestions!


Client Expectations
-------------------

As with systems which supports version negotiation, a robust client
consuming this API will need to also support some range of versions
otherwise that client will not be able to be used in software that
talks to multiple clouds.

The concrete example is nodepool in OpenStack Infra. Assume there is a
world where it is regularly connecting to 4 public clouds. They are
at the following states::

  - Cloud A:
    - min_ver: 2.100
    - max_ver: 2.300
  - Cloud B:
    - min_ver: 2.200
    - max_ver: 2.450
  - Cloud C:
    - min_ver: 2.300
    - max_ver: 2.600
  - Cloud D:
    - min_ver: 2.400
    - max_ver: 2.800

No single version of the API is available in all those clouds based on
the age of some of them. However within the client SDK certain
basic functions like boot will exist, though one might get different
additional data based on the version of the API. The client should smooth
over these differences when possible.

Realistically this is a problem that exists today, except there is no
infrastructure to support creating a solution to solve it.


Alternatives
------------

One alternative is to make all the backwards incompatible changes at
once and do a major API release. For example, change the url prefix to
/v3 instead of /v2. And then support both implementations for a long
period of time. This approach has been difficult in the past and has
cause long periods of time before adoption by various users.

Data model impact
-----------------

None

REST API impact
---------------

As described above there would be additional version information added
to the GET /. These should be backwards compatible changes.

Otherwise there are no changes unless a client header as described is
supplied as part of the request.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

SDK authors will need to start using the X-OpenStack-Cinder-API-Version header
to get access to new features. The fact that new features will only be
added in new versions will encourage them to do so.

python-cinderclient is in an identical situation and will need to be
updated to support the new header in order to support new API
features.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

This will affect how Cinder developers modify the REST API code and add new
extensions.
This will affect the Volume Manager and the ability to remove locks and instead
return VolumeIsBusy for volumes in various -ing states.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Scott DAngelo


Work Items
----------

* Port Manila code for api-microversions
  Status: Done (see https://review.openstack.org/#/c/224910/)
* Add examples of increments and decorators (scottda)
  Status: Done (see https://github.com/scottdangelo/TestCinderAPImicroversions)
* Implement changes for python-cinderclient (scottda)
* test with python-cinderclient changes (scottda)
* Cinder Doc changes (scottda)


Dependencies
============

Any Cinder spec which makes backwards incompatible changes to the API is
dependent on this spec


Testing
=======

It is not feasible for tempest to test all possible combinations
of the API supported by microversions. We will have to pick specific
versions which are representative of what is implemented. The existing
Cinder tempest tests will be used as the baseline for future API
version testing.


Documentation Impact
====================

Documents concerning the API will need to reflect these changes.
These are begun in the WIP cinder code changes, and will live in
cinder/api/openstack/rest_api_version_history.rst


References
==========

* http://git.openstack.org/cgit/openstack/nova-specs/tree/specs/kilo/implemented/api-microversions.rst
* Manila code for api-microversions: https://review.openstack.org/#/c/207228/8
* WIP implementation code: https://review.openstack.org/#/c/224910/
* Test Cases and code: https://github.com/scottdangelo/TestCinderAPImicroversions

..  _OpenStack API Change Guidelines: http://specs.openstack.org/openstack/api-wg/guidelines/evaluating_api_changes.html

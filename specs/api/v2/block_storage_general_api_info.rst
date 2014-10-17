========================================
General Block Storage v2 API information
========================================

OpenStack Block Storage provides volume management with the OpenStack
Compute service.

This document describes the features available with the Block Storage
API v2.

We welcome feedback, comments and bug reports at
`bugs.launchpad.net/Cinder <http://bugs.launchpad.net/cinder>`__.

Intended audience
-----------------

This spec assumes the reader has a general understanding of storage and is familiar with these concepts:

-  ReSTful web services

-  HTTP/1.1 conventions

-  JSON and/or XML data serialization formats

The Block Storage API is implemented using a ReSTful web service
interface. Like other OpenStack projects, Block Storage shares a common
token-based authentication system that allows access between products
and services.

Note
~~~~

All requests to authenticate against and operate the service are
performed using SSL over HTTP (HTTPS) on TCP port 443.

Authentication
--------------

You can use `cURL <http://curl.haxx.se/>`__ to try the authentication
process in two steps: get a token, and send the token to a service.

#. Get an authentication token by providing your user name and either
   your API key or your password. Here are examples of both approaches:

   *You can request a token by providing your user name and your
   password.*

   .. code::

       $ curl -X POST https://localhost:5000/v2.0/tokens -d '{"auth":{"passwordCredentials":{"username": "joecool", "password":"coolword"}, "tenantId":"5"}}' -H 'Content-type: application/json'

   Successful authentication returns a token which you can use as
   evidence that your identity has already been authenticated. To use
   the token, pass it to other services as an ``X-Auth-Token`` header.

   Authentication also returns a service catalog, listing the endpoints
   you can use for Cloud services.

#. Use the authentication token to send a GET to a service you would
   like to use.

Authentication tokens are typically valid for 24 hours. Applications
should be designed to re-authenticate after receiving a 401
(Unauthorized) response from a service endpoint.

Important
~~~~~~~~~

If you programmatically parse an authentication response, be aware that
service names are stable for the life of the particular service and can
be used as keys. You should also be aware that a user's service catalog
can include multiple uniquely-named services that perform similar
functions.


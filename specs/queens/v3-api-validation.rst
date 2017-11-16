..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
API Validation
==============

https://blueprints.launchpad.net/cinder/+spec/json-schema-validation

Currently, Cinder has different implementations for validating
request bodies. The purpose of this blueprint is to track the progress of
validating the request bodies sent to the Cinder server, accepting requests
that fit the resource schema and rejecting requests that do not fit the
schema. Depending on the content of the request body, the request should
be accepted or rejected consistently.


Problem description
===================

Currently Cinder doesn't have a consistent request validation layer. Some
resources validate input at the resource controller and some fail out in the
backend. Ideally, Cinder would have some validation in place to catch
disallowed parameters and return a validation error to the user.

The end user will benefit from having consistent and helpful feedback,
regardless of which resource they are interacting with.


Use Cases
=========

As a user or developer, I want to observe consistent API validation and values
passed to the Cinder API server.


Proposed change
===============

One possible way to validate the Cinder API is to use jsonschema similar to
Nova, Keystone and Glance (https://pypi.python.org/pypi/jsonschema).
A jsonschema validator object can be used to check each resource against an
appropriate schema for that resource. If the validation passes, the request
can follow the existing flow of control through the resource manager to the
backend. If the request body parameters fails the validation specified by the
resource schema, a validation error wrapped in HTTPBadRequest will be returned
from the server.

Example:
"Invalid input for field 'name'. The value is 'some invalid name value'.

Each API definition should be added with the following ways:

* Create definition files under ./cinder/api/schemas/.
* Each definition should be described with JSON Schema.
* Each parameter of definitions(type, minLength, etc.) can be defined from
  current validation code, DB schema, unit tests, Tempest code, or so on.

Some notes on doing this implementation:

* Common parameter types can be leveraged across all Cinder resources. An
  example of this would be as follows::

    from cinder.api.validation import parameter_types
    # volume create schema
    <snip>
         create = {
            'type': 'object',
            'properties': {
                'volume': {
                    'type': 'object',
                    'properties': {
                        'size': parameter_types.positive_integer,
                        'availability_zone': parameter_types.availability_zone,
                        'source_volid': {
                            'format': 'uuid',
                        },
                        'description': parameter_types.description,
                        'multiattach': parameter_types.boolean,
                        'snapshot_id': {
                            'format': 'uuid',
                        },
                        'name': parameter_types.name,
                        'imageRef': {
                            'format': 'uuid',
                        },
                        'volume_type': {
                            'format': 'uuid',
                        },
                        'metadata': {
                            'type': 'object',
                         },
                        'consistencygroup_id': {
                            'format': 'uuid',
                        },
                    },
                'required': ['size'],
                'additionalProperties': False,
            }
            'required': ['server'],
            'additionalProperties': False,
        }
    }

    create_v312 = copy.deepcopy(create)
    create_v312['properties']['volume'][
        'properties']['group_id'] = {'format': 'uuid',},

    parameter_types.py:

    name = {
        'type': 'string', 'minLength': 0, 'maxLength': 255,
    }

    positive_integer = {
        'type': ['integer', 'string'],
        'pattern': '^[0-9]*$', 'minimum': 1, 'minLength': 1
    }

    description = name
    availability_zone = name

    # This registers a FormatChecker on the jsonschema module.
    # It might appear that nothing is using the decorated method but it gets
    # used in JSON schema validations to check uuid formatted strings.
    from oslo_utils import uuidutils

    @jsonschema.FormatChecker.cls_checks('uuid')
    def _validate_uuid_format(instance):
        return uuidutils.is_uuid_like(instance)

    boolean = {
        'type': ['boolean', 'string'],
        'enum': [True, 'True', 'TRUE', 'true', '1', 'ON', 'On', 'on',
                 'YES', 'Yes', 'yes',
                 False, 'False', 'FALSE', 'false', '0', 'OFF', 'Off', 'off',
                 'NO', 'No', 'no'],
    }

* The validation can take place at the controller layer using below decorator::

    from cinder.api.schemas import volume

    @wsgi.response(http_client.ACCEPTED)
    @validation.schema(volume.create, "3.0")
    @validation.schema(volume.create_v312, "3.12")
    def create(self, req, body):
        """creates a volume.

         version 3.12 added groupd_id to the volume request body.
         If user has requested volume create with version v2 then the
         'validation.schema' decorator will ignore its schema validation and
         will pass the request as it is in the create method."""

* Initial work will include capturing the Block Storage API Spec for existing
  resources in a schema. This should be a one time operation for each
  major version of the API. This will be applied to the Block Storage V3 API.

* When adding a new extension to Cinder, the new extension must be proposed
  with its appropriate schema.


Alternatives
------------

Before the API validation framework, we needed to add the validation code into
each API method in ad-hoc. These changes would make the API method code dirty
and we needed to create multiple patches due to incomplete validation.

If using JSON Schema definitions instead, acceptable request formats are clear
and we don’t need to do ad-hoc works in the future.

Pecan Framework:
`Pecan <http://pecan.readthedocs.org/en/latest/>`_
Some projects(Ironic, Ceilometer, etc.) are implemented with Pecan/WSME
frameworks and we can get API documents automatically from the frameworks.
In WSME implementation, the developers should define API parameters for
each API. Pecan would make the implementations of API routes(URL, METHOD) easy.


Data model impact
-----------------

None


REST API impact
---------------

API Response code changes:

There are some occurrences where API response code will change while adding
schema layer for them. For example, On current master 'services' table has
'host' and 'binary' of maximum 255 characters in database table. While updating
service user can pass 'host' and 'binary' of more than 255 characters which
obviously fails with 404 ServiceNotFound wasting a database call. For this we
can restrict the 'host' and 'binary' of maximum 255 characters only in schema
definition of 'services'. If user passes more than 255 characters, he/she will
get 400 BadRequest in response.

API Response error messages:

There will be change in the error message returned to user. For example,
On current master if user passes more than 255 characters for volume name
then below error message is returned to user from cinder-api:

Invalid input received: name has <actual no of characters user passed>
characters, more than 255.

With schema validation below error message will be returned to user for this
case:

Invalid input for field/attribute name. Value: <value passed by user>.
'<value passed by user>' is too long.


Security impact
---------------

The output from the request validation layer should not compromise data or
expose private data to an external user. Request validation should not
return information upon successful validation. In the event a request
body is not valid, the validation layer should return the invalid values
and/or the values required by the request, of which the end user should know.
The parameters of the resources being validated are public information,
described in the Block Storage API spec, with the exception of private data.
In the event the user's private data fails validation, a check can be built
into the error handling of the validator to not return the actual value of the
private data.

jsonschema documentation notes security considerations for both schemas and
instances:
http://json-schema.org/latest/json-schema-core.html#anchor21

Better up front input validation will reduce the ability for malicious user
input to exploit security bugs.


Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Cinder will need some performance cost for this comprehensive request
parameters validation, because the checks will be increased for API parameters
which are not validated now.


Other deployer impact
---------------------

None


Developer impact
----------------

This will require developers contributing new extensions to Cinder to have
a proper schema representing the extension's API.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
Dinesh_Bhor (Dinesh Bhor <dinesh.bhor@nttdata.com>)

Work Items
----------

1. Initial validator implementation, which will contain common validator code
   designed to be shared across all resource controllers validating request
   bodies.
2. Introduce validation schemas for existing core API resources.
3. Introduce validation schemas for existing API extensions.
4. Enforce validation on proposed core API additions and extensions.
5. Remove duplicated ad-hoc validation code.
6. Add unit and end-to-end tests of related API's.
7. Add/Update cinder documentation.

Dependencies
============

The extension's which are there under cinder/api/contrib/ are getting called by
v2 as well as v3. So if we add schema validation for v3 then we will have to
remove the existing validation of parameters which is there inside of
controller methods which will again break the v2 apis.

Solution:

1. Do the schema validation for v3 apis using the @validation.schema decorator
   similar to Nova and also keep the validation code which is there inside of
   method to keep v2 working.

2. Once the decision is made to remove the support to v2 we should remove the
   validation code from inside of method.


Testing
=======

Tempest tests can be added as each resource is validated against its schema.
These tests should walk through invalid request types.

We can follow some of the validation work already done in the Nova V3 API:

* `Validation Testing <http://git.openstack.org/cgit/openstack/tempest/tree/etc/schemas/compute/flavors/flavors_list.json?id=24eb89cd3efd9e9873c78aacde804870962ddcbb>`_

* `Negative Validation Testing <http://git.openstack.org/cgit/openstack/tempest/tree/tempest/api/compute/flavors/test_flavors_negative.py?id=b2978da5ab52e461b06a650e038df52e6ceb5cd6>`_

Negative validation tests should use tempest.test.NegativeAutoTest


Documentation Impact
====================

1. The cinder API documentation will need to be updated to reflect the
   REST API changes.
2. The cinder developer documentation will need to be updated to explain
   how the schema validation will work and how to add json schema for
   new API's.


References
==========

Useful Links:

* [Understanding JSON Schema] (http://spacetelescope.github.io/understanding-json-schema/reference/object.html)

* [Nova Validation Examples] (http://git.openstack.org/cgit/openstack/nova/tree/nova/api/validation)

* [JSON Schema on PyPI] (https://pypi.python.org/pypi/jsonschema)

* [JSON Schema core definitions and terminology] (http://tools.ietf.org/html/draft-zyp-json-schema-04)

* [JSON Schema Documentation] (http://json-schema.org/documentation.html)

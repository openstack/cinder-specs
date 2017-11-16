======
Faults
======

When an error occurs, Block Storage returns a fault object containing an
HTTP error response code that denotes the type of error. The response
body returns additional information about the fault.

The following table lists possible fault types with their associated
error codes and descriptions.

=======================  =====================  ===============================
Fault type               Associated error code  Description
=======================  =====================  ===============================
``badRequest``           400                    The user request contains one
                                                or more errors.
``unauthorized``         401                    The supplied token is not
                                                authorized to access the
                                                resources, either it's expired
                                                or invalid.
``forbidden``            403                    Access to the requested
                                                resource was denied.
``itemNotFound``         404                    The back-end services did not
                                                find anything matching the
                                                Request-URI.
``badMethod``            405                    The request method is not
                                                allowed for this resource.
``overLimit``            413                    Either the number of entities
                                                in the request is larger than
                                                allowed limits, or the user has
                                                exceeded allowable request rate
                                                limits. See the ``details``
                                                element for more specifics.
                                                Contact your cloud provider if
                                                you think you need higher
                                                request rate limits.
``badMediaType``         415                    The requested content type is
                                                not supported by this service.
``unprocessableEntity``  422                    The requested resource could
                                                not be processed on at the
                                                moment.
``instanceFault``        500                    This is a generic server error
                                                and the message contains the
                                                reason for the error. This
                                                error could wrap several error
                                                messages and is a catch all.
``notImplemented``       501                    The requested method or
                                                resource is not implemented.
``serviceUnavailable``   503                    Block Storage is not available.
=======================  =====================  ===============================


The following two ``instanceFault`` examples show errors when the server
has erred or cannot perform the requested operation:

**Example 2.4. Example instanceFault response: XML**

.. code::

    HTTP/1.1 500 Internal Server Error
    Content-Type: application/xml
    Content-Length: 121
    Date: Mon, 28 Nov 2011 18:19:37 GMT

.. code::

    <?xml version="1.0" encoding="UTF-8"?>
    <instanceFault code="500"
        xmlns="http://docs.openstack.org/openstack-block-storage/2.0/content">
        <message> The server has either erred or is incapable of
            performing the requested operation. </message>
    </instanceFault>

|

**Example 2.5. Example fault response: JSON**

.. code::

    HTTP/1.1 500 Internal Server Error
    Content-Length: 120
    Content-Type: application/json; charset=UTF-8
    Date: Tue, 29 Nov 2011 00:33:48 GMT

.. code::

    {
       "instanceFault":{
          "code":500,
          "message":"The server has either erred or is incapable of performing the requested operation."
       }
    }

|

The error code (``code``) is returned in the body of the response for
convenience. The ``message`` element returns a human-readable message
that is appropriate for display to the end user. The ``details`` element
is optional and may contain information that is useful for tracking down
an error, such as a stack trace. The ``details`` element may or may not
be appropriate for display to an end user, depending on the role and
experience of the end user.
`
The fault's root element (for example, ``instanceFault``) may change
depending on the type of error.

The following two ``badRequest`` examples show errors when the volume
size is invalid:

**Example 2.6. Example badRequest fault on volume size errors: XML**

.. code::

    HTTP/1.1 400 None
    Content-Type: application/xml
    Content-Length: 121
    Date: Mon, 28 Nov 2011 18:19:37 GMT

.. code::

    <?xml version="1.0" encoding="UTF-8"?>
    <badRequest code="400"
        xmlns="http://docs.openstack.org/openstack-block-storage/2.0/content">
        <message> Volume 'size' needs to be a positive integer value, -1.0
            cannot be accepted. </message>
    </badRequest>

|

**Example 2.7. Example badRequest fault on volume size errors: JSON**

.. code::

    HTTP/1.1 400 None
    Content-Length: 120
    Content-Type: application/json; charset=UTF-8
    Date: Tue, 29 Nov 2011 00:33:48 GMT

.. code::

    {
       "badRequest":{
          "code":400,
          "message":"Volume 'size' needs to be a positive integer value, -1.0 cannot be accepted."
       }
    }

|

The next two examples show ``itemNotFound`` errors:

**Example 2.8. Example itemNotFound fault: XML**

.. code::

    HTTP/1.1 404 Not Found
    Content-Length: 147
    Content-Type: application/xml; charset=UTF-8
    Date: Mon, 28 Nov 2011 19:50:15 GMT

.. code::

    <?xml version="1.0" encoding="UTF-8"?>
    <itemNotFound code="404"
        xmlns="http://docs.openstack.org/api/openstack-block-storage/2.0/content">
        <message> The resource could not be found. </message>
    </itemNotFound>

|

**Example 2.9. Example itemNotFound fault: JSON**

.. code::

    HTTP/1.1 404 Not Found
    Content-Length: 78
    Content-Type: application/json; charset=UTF-8
    Date: Tue, 29 Nov 2011 00:35:24 GMT

.. code::

    {
       "itemNotFound":{
          "code":404,
          "message":"The resource could not be found."
       }
    }

|


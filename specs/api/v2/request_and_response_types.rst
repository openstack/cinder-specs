==========================
Request and response types
==========================

The Block Storage API supports both the JSON and XML data serialization
formats. The request format is specified using the ``Content-Type``
header and is required for calls that have a request body. The response
format can be specified in requests either by using the ``Accept``
header or by adding an ``.xml`` or ``.json`` extension to the request
URI. Note that it is possible for a response to be serialized using a
format different from the request. If no response format is specified,
JSON is the default. If conflicting formats are specified using both an
``Accept`` header and a query extension, the query extension takes
precedence.

Some operations support an Atom representation that can be used to
efficiently determine when the state of services has changed.

**Table 2.1. Response formats**

======           =============    =============== =======
Format           Accept Header    Query Extension Default
======           =============    =============== =======

JSON            application/json  .json           Yes

XML             application/xml   .xml            No

In the request example below, notice that *``Content-Type``* is set to
*``application/json``*, but *``application/xml``* is requested via the
*``Accept``* header:

**Example 2.1. Request with headers: Get volume types**

.. code::

    GET /v2/441446/types HTTP/1.1
    Host: dfw.blockstorage.api.openstackcloud.com
    X-Auth-Token: eaaafd18-0fed-4b3a-81b4-663c99ec1cbb
    Accept: application/xml

|

An XML response format is returned:

**Example 2.2. Response with headers**

.. code::

    HTTP/1.1 200 OK
    Date: Fri, 20 Jul 2012 20:32:13 GMT
    Content-Length: 187
    Content-Type: application/xml
    X-Compute-Request-Id: req-8e0295cd-a283-46e4-96da-cae05cbfd1c7

    <?xml version='1.0' encoding='UTF-7'?>
      <volume_types>
          <volume_type id="1" name="SATA">
              <extra_specs/>
          </volume_type>
          <volume_type id="2" name="SSD">
              <extra_specs/>
          </volume_type>
      </volume_types>

|


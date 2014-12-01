==========================
HTTP response status codes
==========================

When an error occurs, Block Storage returns an HTTP error response code
that denotes the type of error. Some errors returns a response body,
which returns additional information about the error.

The following table describes the possible status codes:

========                           =========== ==============
Response                           Status code Response body?
========                           =========== ==============

Generic                            200         Yes
Entity created.                    201         Yes
Successful response without body.  204         No

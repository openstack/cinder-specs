======
Limits
======

All accounts, by default, have a preconfigured set of thresholds (or
limits) to manage capacity and prevent abuse of the system. The system
recognizes two kinds of limits: *rate limits* and *absolute limits*.
Rate limits are thresholds that are reset after a certain amount of time
passes. Absolute limits are fixed.

Rate limits
~~~~~~~~~~~

Rate limits are specified in terms of both a human-readable wild-card
URI and a machine-processable regular expression. The regular expression
boundary matcher '^' takes effect after the root URI path. For example,
the regular expression ^/v1.0/instances would match the bolded portion
of the following URI:
https://dfw.blockstorage.api.openstackcloud.com\ **/v1.0/instances**.

The following table specifies the default rate limits for all API
operations for all **GET**, **POST**, **PUT**, and **DELETE** calls for
volumes:

**Table 2.2. Default rate limits**

Verb

URI

RegEx

Default

**GET** changes-since

\*/instances/\*

^/v\\d+\\.\\d+/\\d+/instances.\*

3/minute

**POST**

\*/instances/\*

^/v\\d+\\.\\d+/\\d+/instances.\*

10/minute

**POST** instances

\*/instances/\*

^/v\\d+\\.\\d+/\\d+/instances.\*

50/day

**PUT**

\*/instances/\*

^/v\\d+\\.\\d+/\\d+/instances.\*

10/minute

**DELETE**

\*/instances/\*

^/v\\d+\\.\\d+/\\d+/instances.\*

100/minute

|

Rate limits are applied in order relative to the verb, going from least
to most specific. For example, although the threshold for **POST** to
/v1.0/\* is 10 per minute, one cannot **POST** to /v1.0/\* more than 50
times within a single day.

If you exceed the thresholds established for your account, a 413 (Rate
Control) HTTP response will be returned with a ``Retry-After`` header to
notify the client when it can attempt to try again.

Absolute limits
~~~~~~~~~~~~~~~

The following table shows the absolute limits:

**Table 2.3. Absolute limits**

Name

Description

Limit

Block Storage

Maximum amount of block storage

1 TB

|


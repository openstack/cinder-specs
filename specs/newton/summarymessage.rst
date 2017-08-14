..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
User facing "Failure" Message and Event Viewer
==============================================

For quite some time, OpenStack services have wanted to be able to send
messages, especially error messages, to API end users (by user I do not mean
the operator, but the user that is interacting with the API).

If user performs some operation and operation fails or goes in
hang state, then there must be some interface for user to see reason
for this behavior.

So, this is basically regarding facilitating user to see error messages
for asynchronous operations either directly through APIs or through new
"Event Viewer" tab in horizon or horizon pluggin.

This is more specific to failed operations and cause of their failure.

https://blueprints.launchpad.net/cinder/+spec/summarymessage

Problem description
===================

If operations like create volume, create snapshot etc fails, user gets
no detailed information in case operation fails.
Sometime only operation status is updated to error, failed etc without
any update to user.

In some case it is worse than this.
For ex,
1. In case rabbitmq is inactive and user tries to create volume,
horizon hangs on waiting response from API forever.
2. In case rabbitmq is active but recipient service is not running
and user tries to create volume, API hangs on waiting response
from rabbitMQ forever.

A few resources among all the OpenStack projects handle reporting errors
to end users for asynchronous operations and those that do are inconsistent
with each other.
In addition to a mechanism to enable error reporting in a
consistent way across OpenStack, the solution must also be able to
accommodate a deployment of Cinder that contains no other OpenStack service.

Use Cases
=========

Motivation for this Blue Print:

1. To help admin in debugging failed operations with
   less log filtering.
2. To notify admin to start services which failed due to
   some abnormal conditions.
3. To inform user about failure in case operation fails.
4. To provide enough information to user about failure.

General Use Cases:

* Cinder volume/snapshot creation goes to ERROR status due
  to lack of capacity. (Scheduling error)
* Cinder add volume to CG fails, how to tell user the volume
  and group are not on the same backend?
* Cinder volume goes from attaching to available. why?
* Volume retype fails.
* Volume extend fails, I'd like to know why and be able to
  still use my volume instead of it being in error_extending.
* ETC..

Proposed change
===============

User can fetch operation failure details through direct API calls or
through new horizon tab "Event Viewer".
From CLI, cli client could be used to display same kind of information.

Suggested implementation is based on 2 way approach

1. Push Information:
   During user operations, messages will be pushed to database
   using component specific notifications.

   These messages will be generated and pushed to database by
   component for operation start, operation completion or operation
   failure.

   Message generation will be based on eventID to eventMessage
   mapping using message constant files which keeps different
   notification messages mapping in it.
   This way deployer can easily modify notification messages as per
   requirement.

2. Pull Information:
   In case user needs to check operation status, user can pull details
   using CLI client or Horizon tab.

Results may be shown in tabular way as shown below

+--------+---------+----------+-------+-------+----------+------+---------+
| Tenant | EventID | NodeName | ReqID | Level | Resource | Time | Summary |
+========+=========+==========+=======+=======+==========+======+=========+
| Sheel  | UKN_ERR | BS-cind1 | {...} | Error | Volume   | {..} | {....}  |
+--------+---------+----------+-------+-------+----------+------+---------+

Every 'operation' initiated by the user has a request ID returned as
an HTTP header in the context.
These notification messages will be tied to operation request ID.
(This request ID will be used for mapping a request to what happened in cinder
for that operation.)

Summary message will contain operation specific failure message.
For ex,
"Volume create operation failed - {Reason of failure}"


Filters:

Results can be filtered depending upon TenantID, HostID, UserID,
Operation Outcome/Result(Fail/Pass) etc.

Type of Messages:

1. API Events : messages for failed operations.
2. Service Logs: information for failed services either stopped by user
   or stopped due to any abnormal conditions.
3. System Logs: any other logs than API and service logs.

**Suggested Architecture**:

The proposed change is to add a new /v3/<tenant>/messages API
resource backed by a messages table in the Cinder DB.
This endpoint will return a list of error messages that are
intended for the end-user regarding failed asynchronous operations.

In short:
* /v3/<tenant>/messages API resource, exposes notifications messages
depending upon filters
* message_ttl config option that dictates message minimum life in seconds
* messages DB table

Questions
---------
None

Alternatives
------------

* User facing notifications
  Use the existing notification framework in combination with an AMQP
  consumer to pull messages off and provide an endpoint for the user.
  Faults with this approach are that we do not want to display the current
  information in notifications to the user and it will require many more
  services as dependencies.

* Per resource faults
  This alternative suggests adding a sub-resource to each resource, such as
  volumes/<volume-id>/faults, similar to Nova's instance faults. This
  makes it difficult to poll for messages for more than a single resource
  or resource type. It also adds significant complexity to the api as
  every resource must add /faults in order to support messages.

* Exposing user messages via a separate service (such as Zaqar)
  This approach suggests storing user messages in another service that the
  user could query for messages or the service could utilize webhooks to
  notify the user. One major drawback to this approach is the complexity
  in writing bindings for the separate service(s) and the need for a
  separate service as a dependency.

What this specification does not solve
--------------------------------------

* State change notifications.
  This solution does not intend to solve the use-case of alerting users when
  a volume or any other resource changes state. For example, when a volume
  changes from ``creating`` to ``available``.

REST API impact
---------------

New APIs:
* GET /v3/<tenant>/messages
With filters by attribute. Ex: GET /v3/<tenant>/messages?resource_type=volume
* GET /v3/<tenant>/messages/<message-id>
* DELETE /v3/<tenant>/messages/<message-id>

Message schema ::

  Message:
    type: object
    required:
    - user_message
    - id
    - project_id
    - request_id
    - event_id
    - created_at
    - message_level
    - expires_at
    properties:
      id:
        type: string
        description: UUID will be stored in 'id' field.
      message_level:
        type: string
        enum:
        - ERROR
        description: The level of the message. In the future we may expand to
        sending information to the user that is not an error.
      user_message:
        type: string
      event_id:
        type: string
        description: Event ID can be used to
        a. update message text at deployer end for some specific situation.
        b. to report errors by user.
        c. to debug fast as it is easy to search where specific eventID is
        used for reporting error.
      resource_uuid:
        type: string
        description: The uuid of the offending resource.
      resource_type:
        type: string
        description: The type of resource this message pertains to.
        For ex, volume, snapshot, backup etc
      request_id:
        type: string
      created_at:
        type: string
      expires_at:
        type: string
        description: After this time the message may no longer exist

Data model impact
-----------------

New messages table in the DB to store all messages. This table may prove to
grow large in a cloud with lots of errors. The admin will be able to utilize
the ``expires_at`` column to reap messages.

Security impact
---------------

Messages must be highly scrutinized before becoming visible to the user in
order to avoid any sensitive data from being shown. This will be mitigated by
having all user visible messages defined in a single module. The messaging
mechanism will assert that any message it will create comes from the sanctioned
location.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

* New configuration option ``message_ttl`` that will dictate the number of
  seconds after the messages creation time to set the 'guaranteed_until'
  attribute on generated messages.

* New configuration option ``message_reap_interval`` that will dictate the
  number of seconds between calls to delete old messages. A value of -1
  will never run. DocImpact: This option should not be set on a large number
  of nodes, since too many nodes trying this delete at the same time will cause
  transaction bouncing and degraded DB performance.

* New configuration option ``message_reap_batch_size`` that dictates the number
  of expired messages to delete each interval. This allows a deployer to
  limit DB performance impact by setting a ceiling for the number of
  messages deleted at a time.

* The messages table will be potentially large and may be reaped based on
  the 'guaranteed_until' column. Where all messages with a
  ``expires_at`` date earlier than the current time can be safely
  deleted.

Developer impact
----------------

Developers should be aware of use-cases where the user needs information
about an error. In these situations, an appropriate user message should be
written and creation of the message added in the specific code path(s).

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sheel Rana
  Alex Meade

Work Items
----------

This whole implementation depends upon message generation, transport,
collection, storage and analytics of different failure messages.

* cinder:
  Implementation to generate notification messages at the time of
  failure for all existing operations.

* cinder:
  notification listener is required which will serve as
  basis for handling event messages from different components.

* cinder:
  collector is required to collect, validate and store event
  messages to database.

* cinder:
  new API to fetch details form database depending upon filters.

* cinder:
  Add pagination to messages

* cinder-manage:
  Add mechanism to automatically, and via a cinder-manage command,
  reap expired messages in the db depending upon ttl value.

* cinder:
  Documentation for new API details.

* cinder:
  Update "Getting started Guide".

* cinder:
  Database schema preparation to store notification messages.

* cinder:
  Need to implement "delete messages as per message life" from database after
  message expiry time.
  For ex, if user has set ``message_ttl`` to 7 days, then all messages
  older than 7 days will be purged from database.

* horizon:
  Separate tab for cinder to display event messages.

* cinder-client:
  cinder cli to communicate with API and fetch event messages.

* cinder-client:
  Update to CLI reference Guide.

* Tempest tests


Implementation Phases:
----------------------
This whole feature will be implemented in multiple phases:

Phase 1. Basic implementation regarding notification generation and storage
into database with "/messages" exposed to view notification messages.
This spec targets Phase 1 first, other phases will be implemented after
acceptance of phase 1.

Phase 2. Implementation for facilitating admin to configure notification
storage like db or zaqar or both.
If both RPC/DB are configured by admin, notification message would be
stored in zaqar along with storing information to database.

Phase 3. Implementation for consuming information from zaqar directly.

Phase 4. Horizon and CLI implementations to view notifications in more
formatted manner.

Phase 5. Handling of some special cases where generation of notifications
requires separate handling like rabbitMQ related implementations for showing
notifications in case rabbitMQ is in failed state or rabbitMQ recipient is
in inactive state.

Dependencies
============
None


Testing
=======

Tempest tests should be written and run in the gate. It may prove difficult
to implement complete functional testing of the feature as messages will not
be created unless there is an error, which may be difficult to trigger.
However, some operations are easy to trigger failure with unlimited quotas.
One example is creating a thick provisioned volume too big to be stored on the
backend.

Example Test Cases
------------------

# List messages with no messages
# Attempt creation of a TOO LARGE volume and verify appropriate scheduling
error message is created
# List messages with filters, especially resource_type

Documentation Impact
====================

* REST API documentation
* New config option, ``message_ttl`` (time to live)
* New config option, ``message_reap_interval``
  (number of seconds between calls to delete old messages)
* New config option, ``message_reap_batch_size``
  (number of messages which could be deleted in one batch)
* New API policies for messages

References
==========

Mitaka Midcycle discussion
 https://etherpad.openstack.org/p/mitaka-cinder-midcycle-user-notifications
 https://etherpad.openstack.org/p/mitaka-cinder-midcycle-day-1

Kilo Summit Discussion
 https://etherpad.openstack.org/p/kilo-cinder-async-reporting

Liberty Summit Discussion (in conjunction with HEAT) -
 https://etherpad.openstack.org/p/liberty-cross-project-user-notifications

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Explicit user messages
======================

https://blueprints.launchpad.net/cinder/+spec/better-user-message

Use 'action', 'resource', 'message' to replace 'event' in user messages.

Problem description
===================

We have introduced user messages since Ocata, that's a beneficial feature,
because it solves the problem that we don't provide enough message when an
async task is failed (especially for volume and snapshot resources).
It's easy to find out when and where the task failed with our new
messages APIs.

But, the better user experience all depend on whether we have hard coded
enough user messages in our projects. So that raises a new question, how could
we add these messages without changing our code frequently and how could we
maintain this 'better user experience but not functionality involved' code
cleanly.

Here are some of our user message related codes:

 .. code-block:: python

    # CODE1: defined event ids
    UNKNOWN_ERROR = 'VOLUME_000001'
    UNABLE_TO_ALLOCATE = 'VOLUME_000002'
    ATTACH_READONLY_VOLUME = 'VOLUME_000003'
    IMAGE_FROM_VOLUME_OVER_QUOTA = 'VOLUME_000004'

 .. code-block:: python

    # CODE2: try/except and write the user messages
    with excutils.save_and_reraise_exception():
        if isinstance(error, exception.ImageLimitExceeded):
            self.message_api.create(
                context,
                defined_messages.EventIds.IMAGE_FROM_VOLUME_OVER_QUOTA,
                context.project_id,
                resource_type=resource_types.VOLUME,
                resource_uuid=volume_id)

We would possibly have several issues if we try to add more user messages
based on those presets.

* [CODE1]: We would redefine a lot of similar event ids, such as
  ``UNABLE_CREATE_VOLUME_DRIVER_NOT_READY``,
  ``UNABLE_CREATE_SNAPSHOT_DRIVER_NOT_READY`` and
  ``UNABLE_CREATE_BACKUP_DRIVER_NOT_READY``, it's obvious that they are all
  due to the same error and are created at same procedure (create resource).

* [CODE2]: To have an explicit message, we would have a bunch of 'if/else'
  codes which only focus on detecting the error class and write down the
  corresponding messages, and even worse if the exception gets more and more,
  we have to update the code again and again.

So could we have a better way to fix these?

Use Cases
=========

The main use-case here is for an explicit user message to end users and
a better maintaining experience for cinder developers.

Proposed change
===============

The proposed changes are:

* Introduce ``message`` (or name it ``detail`` to avoid confusion) to user
  messages, the 'message' only represent what (no matter error or warning)
  happened at that explicit time, that means ``QUOTA_EXCEED`` is a valid one
  while ``UPLOAD_TO_IMAGE_OVER_QUOTA`` isn't.

* Build the relationship from exceptions to messages, we have a bunch of
  exceptions which could directly point out what happened and how to fix,
  but we couldn't expose this to the end user because it would expose some
  sensitive information. We couldn't hard code the cleaned messages into
  exceptions either, because the exceptions would be defined at different
  places (take cinder exceptions and driver exceptions for example) and
  different exceptions can be translated into one user messages. so this
  is proposed how to do, build a dictionary from exception **name** to
  ``messages`` (introduced above)::

     # multi_key dictionary
     ['HBSDBusy','XtremIOArrayBusy']: MessageIds.DEVICE_IS_BUSY
     ['ConfigNotFound',
      'ViolinInvalidBackendConfig']: MessageIds.INVALID_CONFIGURATION

     message_map = {
     MessageIds.DEVICE_IS_BUSY: _("Backend device is busy at present."),
     MessageIds.INVALID_CONFIGURATION: _("Invalid configuration."),
     }
     ...

* Introduce ``action`` to user messages. This one should be introduced along
  with the ``messages``, and can use it to indicate when the message is
  created, such as ``CREATE_SNAPSHOT_IN_BACKEND``, ``UPLOAD_TO_IMAGE``.

* Generate ``event_id`` automatically, user message is used to build user
  friendly messages for project which directly display the information for
  administrators. But when referring to the projects which detect, classify
  and use our messages, the event_id is more useful at this case, so we should
  have 'event_id' no matter how we change the underground behaviour. Instead
  of manually define and maintain this event_ids we could build them by a
  combination of ``RESOURCE``, ``ACTION`` and ``MESSAGE``, take these for
  example::

    # This is how we define event currently
    IMAGE_FROM_VOLUME_OVER_QUOTA = 'VOLUME_000004'

    # This is how we would build the event_id at response after this change.
    VOLUME_BACKUP_001_0002 : ({PROJECT}_{RESOURCE}_{ACTION_ID}_{MESSAGE_ID})

  We could have these benefits:

  1. We don't need to define that much events. (we only need to
     define less messages).

  2. It's also unique cross all of OpenStack.

  3. It's reading friendly and easy to classify or analysis.

Alternatives
------------

The alternatives is we could keep adding user messages in the way of we
currently have `[1]`_. (There could be more alternatives or better solutions,
but I failed to figure out.)

Data model impact
-----------------

Database update is required to store the 'action_id' and 'message_id', also
we can deprecate the 'event_id', because we could generate it anytime we
want.

REST API impact
---------------

We don't have API impact because we didn't expose the create API.

Security impact
---------------

None

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

None

Developer impact
----------------

For a better experience, developers have to maintain the relationship
from exception to messages when any related change is made.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Upgrade database to reflect new user message object.
* Support 'exception' and 'action' in user message APIs.
* Add some unit tests.
* Add script for database migration.

Dependencies
============

None


Testing
=======

* Unit tests

Documentation Impact
====================

* Update developer documentation for 'exception-to-message' dictionary.

References
==========

_`[1]` https://review.openstack.org/#/c/298052/

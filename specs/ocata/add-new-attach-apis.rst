..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Introduce new volume attach APIs
==========================================
https://blueprints.launchpad.net/cinder/+spec/add-new-attach-apis


Problem description
===================

The current volume attach process for Cinder Volumes is somewhat inconsistent
and rather unclear in terms of exactly how it works.  Some of this has been
a result of growth and feature adds, things like optional parameters have made
the process a bit more complex than it needs to be.

The interpretation of things like initialize_connection has sort of diverged a
bit, and due to some requirements for different backends things have become a
bit hacky.  We have the potential for race conditions during the attach/detach
process and most of all we seem to have a situation where adding things like
multi-attach support are extremely complex and difficult.

Rather than continuing to add on to the monolith, it might be a good time to
step back and simplify the flow altogether.

Use Cases
=========

Attaching Cinder volumes to Nova Instances, Bare Metal or whatever.

Proposed change
===============

Rather than using reserve, initialize and attach with numerous parameters to
chain together an attach, and try to modify things to make multi-attach work
this specs proposes that we instead introduce a new simplified and more robust
set of attach API's.

API Operations:

* attachment_create

  This call is used to create a volume attachment.

  If the connector is not specified this call will only complete the volume
  reservation step and create an empty volume attachment. This can be used
  by Nova to call it from the API and start the volume attaching process by
  checking that the volume is available and creates the empty attachment. To
  finalize the operation Nova will call 'attachment_update'.

  In case the connector is specified it will reserve the volume and create the
  connection between the volume and the host and update the attachment record.
  This can be used for instance for bare metal use cases, when the connector is
  available immediately.

  In both cases the call returns the attachment_id to the caller.

  Args:
      volume_id
      instance_uuid
      connector (optional)
  Returns:
      attachment_id

* attachment_update

  This call is used to update an existing volume attachment. For this call the
  connector is mandatory.

  In case of an empty attachment it also finalizes attaching the volume and
  updates the attachment record accordingly.

  As the attachment exists in every scenario this call is invoked the
  attachment_id is mandatory.

  Args:
      attachment_id
      connector
  Returns:
      connection_info

* attachment_get

  This call will fetch the specified attachment object (by ID).  In addition
  this call will go out to the Cinder backend driver and query it for any
  shared connection info.  The purpose being that we return the attachment
  object that was requested as well as a list of any shared attachment_id's.

  Args:
      attachment_id
  Returns:
      Attachment info

      {id: <uuid-of-the-attachment-object>,
       project_id: <project-id-associated-with-attachment>,
       volume_id: <uuid-of-volume-associated-with-this-attachment-record>,
       instance_uuid: <uuid-of-instance-attachment-is-used-for>,
       attached_host: <hostname-of-compute-node-attached-to>,
       mountpoint: <mountpoint-inside-instance-that-is-expected-to-be-used>,
       attach_time: <time-we-created-the-attachment>,
       detach_time: <time-we-detached>,
       attach_status: <status-of-the-attachment>,
       attach_mode: <ro|rw>,
       shared_connections_by_attachment: [<attachment_id>, ...]}

* attachment_remove

  Removes/Detaches an existing attachment, marks a volume as available if no
  more attachments to it left.

  Args:
      attachment_id
  Returns:
      List of attachment_id's for any shared connections

Alternatives
------------

Keep trying to hack our way around the existing attach/detach flows we have.
Deal with the split brain issue we have between Nova and Cinder.

It is possible that we could rewrite the existing methods and enforce stricter
interface usage and we'd probably be ok, but the code has become pretty
convoluted and ugly to deal with and it might be easier to wrap the existing
calls up into a managing wrapper and then over time iterate over the internal
pieces.

Data model impact
-----------------

We need to store the connector when making attachments. As the connector can
vary based on the back end and can also easily exceed 255 chars in length
adding it to the 'volume_attachment' table is not a viable option. As opposed
to that we'll add an 'attachment_specs' table that will slurp in the k/v
entries in the connector.

REST API impact
---------------

This will introduce at least the 4 new calls mentioned in the proposal section,
these will be part of a new API object named `attachment`.  We'll treat the
attachment as a first class object in Cinder.

The new API will be released under a new microversion.

Security impact
---------------

Notifications impact
--------------------

Other end user impact
---------------------

The old API's will still work until we decide to deprecate them.

Performance Impact
------------------

Other deployer impact
---------------------

Developer impact
----------------

For now this is avoiding changes to the drivers in Cinder and focusing on
just the API for the attach/detach call up until the Manager layer.

Implementation
==============

Condense initialize_connection and attach down into a single attachment_create
call.

The intent here is that Cinder shouldn't know or care so much about what a
consumer is doing with a volume or attachment.  We should only care that they
desire to attach or detach and that's about it.  In order to do this properly
we will need an attachment_id that the consumer should store and reference
going forward when they are done with the attachment.  This means it's up to
the consumer to decide when they are truly done and they want to destroy the
attachment as opposed to now where we try and guess for them based on clues.



Assignee(s)
-----------

Primary assignee:
  john-griffith

Work Items
----------

1. Implement API calls and wrappers in Cinder

2. Implement cinderclient calls to expose them

3. Implement changes in Nova

Dependencies
============


Testing
=======

Once the API design is finalized the same Tempest test coverage will be added
as we have for the current attach/detach API. In parallel to this activity the
Nova implementation to pick up the new API shall be started as well which will
add an additional layer of testing.

Documentation Impact
====================

Will need to update any and all documents about how attach/detach of
volumes in Cinder works.

References
==========

 * `Nova spec`_

.. _Nova spec: https://review.openstack.org/#/c/373203/

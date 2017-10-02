..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Use the enginefacade from oslo_db
==================================

https://blueprints.launchpad.net/cinder/+spec/use-oslodb-enginefacade

Implement the new oslo.db enginefacade interface described here:

https://blueprints.launchpad.net/oslo.db/+spec/make-enginefacade-a-facade


Problem description
===================

The linked oslo.db spec contains the details of the proposal, including its
general advantages to all projects. In summary, we transparently track database
transactions using the cinder.RequestContext object. This means that if there
is already a transaction in progress we will use it by default, only creating a
separate transaction if explicitly requested.

Use Cases
=========

These changes will only affect developers.

* Allow a class of database races to be fixed

Cinder currently only exposes db transactions in cinder/db/sqlalchemy/api.py,
which means that every db api call is in its own transaction. Although this
will remain the same initially, the new interface allows a caller to extend a
transaction across several db api calls if they wish. This will enable callers
who need these to be atomic to achieve this, which includes the save operation
on several Cinder objects.

* Reduce connection load on the database

Many database api calls currently create several separate database connections,
which increases load on the database. By reducing these to a single connection,
load on the db will be decreased.

* Improve atomicity of API calls

By ensuring that database api calls use a single transaction, we fix a class of
bug where failure can leave a partial result.

* Make greater use of slave databases for read-only transactions

The new api marks sections of code as either readers or writers, and enforces
this separation. This allows us to automatically use a slave database
connection for all read-only transactions. It is currently only used when
explicitly requested in code.

Proposed change
===============

* Decorate the cinder.RequestContext class

cinder.RequestContext is annotated with the
@enginefacade.transaction_context_provider decorator. This adds several code
hooks which provide access to the transaction context via the
cinder.RequestContext object, including attributes 'session', 'connection',
'transaction'.

* Ensure that all database apis use context session

Use context session in cinder.db.sqlalchemy.api.model_query().

* Update database apis incrementally

Individual calls will be annotated as either readers or writers. Existing
transaction management will be replaced.

Database apis will be updated in batches, by function. For example, Volume
apis, Snapshot apis, Quota apis. Calls into apis which have not been upgraded
yet will continue to explicitly pass the session or connection object.

Alternatives
------------

Alternatives were examined during the design of the oslo.db code. The goal
of this change is to implement a solution which is common across OpenStack
projects.

Data model impact
-----------------

None

REST API impact
---------------

None

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

By reducing connection load on the database, the change is expected to provide
a small performance improvement. However, the primary purpose is correctness.

Other deployer impact
---------------------

None


Developer impact
----------------

The initial phase of this work will be to implement the new engine facade in
cinder/db/sqlalchemy/api.py only. Callers will not have to consider transaction
context if they do not currently do so, as it will be created and destroyed
automatically.

This change will allow developers to explicitly extend database transaction
context to cover several database calls. This allows the caller to make
multiple database changes atomically.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Hanxi_Liu <hanxi.liu@easystack.cn>


Work Items
----------

* Enable use of the new api in Cinder

* Migrate api bundles along functional lines:
    * Backup, BackupMetadata
    * Cluster
    * ConsistencyGroup, Cgsnapshot
    * DriverInitiatorData
    * Encryption
    * Group, GroupSnapshot, GroupTypes, GroupTypeSpecs,
      GroupTypeProjects, GroupVolumeTypeMapping
    * ImageVolumeCacheEntry
    * Message
    * QualityOfServiceSpecs
    * Quota, QuotaClass, QuotaUsage, Reservation
    * Service
    * Snapshot, SnapshotMetadata
    * Transfer
    * VolumeAttachment, AttachmentSpecs
    * Volume, VolumeMetadata, VolumeAdminMetadata, VolumeGlanceMetadata
    * VolumeTypes, VolumeTypeProjects, VolumeTypeExtraSpecs


Dependencies
============

A version of oslo.db including the new enginefacade api:

https://review.openstack.org/#/c/138215/

Testing
=======

This change is intended to have no immediate functional impact. The current
tests should continue to pass.

Documentation Impact
====================

None

References
==========

* Implement the oslo.db enginefacade interface described here:

https://blueprints.launchpad.net/oslo.db/+spec/make-enginefacade-a-facade

* Use the new oslo_db enginefacade in Nova:

https://blueprints.launchpad.net/nova/+spec/new-oslodb-enginefacade

* Use the new oslo_db enginefacade in Neutron:

https://blueprints.launchpad.net/neutron/+spec/enginefacade-switch

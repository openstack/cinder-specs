..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
User visible information in volume types
========================================

https://blueprints.launchpad.net/cinder/+spec/user-visible-information-in-volume-types

Volume types are used to find a fitting backend when creating a volume. But
they also imply a few configurations to the volume to be created, that are
good to know for anyone who has to choose from various volume types.

Problem description
===================

When a non-admin user is creating a volume they would like to see various
information about the volume type. Some of those information are already
visible for users, when looking into a volume type (e.g. multiattach). Other
information are either not available at all or not visible for non-admin
users. This is a problem when trying to choose a fitting volume type when
creating a volume.

Use Cases
=========

The information users would like to see are first and foremost:
1. Whether a volume type can be used to create encrypted volumes
2. Whether a volume type creates replicated volumes, either through Cinder or
through a configured backend like ceph

The first information comes from the encryption type within the volume, which
is only accessible for an admin. The second information can either be set and
directly seen in the volume type extra_specs or is indirectly a part of the
configuration of the backend.

There might be more use cases, that should benefit from a solution allowing
an admin to add certain user facing information into the volume type in a
uniform way.

As there are more and more tools that automatically create IaaS resources,
that information should be accessible in a uniform way and could best be used
to filter for fitting volume types in the client.

Proposed change
===============

To let an operator add information that is visible for users and also machine-
readable, we introduce a new field in the volume type object called "metadata".
This will include API changes to address the new view of the volume type as
well as a new API endpoint to set information as key value pairs for this
metadata field. A new database table for those metadata is also necessary.

The table will consist of columns for the volume type id, the key, the value,
an id as the primary key and metadata for creation, updates and deletion. The
upgrade path for the database may look like this:

.. code-block:: python

  def upgrade():
    op.create_table(
        'volume_type_metadata',
        sa.Column('created_at', sa.DateTime),
        sa.Column('updated_at', sa.DateTime),
        sa.Column('deleted_at', sa.DateTime),
        sa.Column('deleted', sa.Boolean),
        sa.Column('id', sa.Integer, primary_key=True, nullable=False),
        sa.Column(
            'volume_type_id',
            sa.String(36),
            sa.ForeignKey(
                'volume_types.id',
                name='volume_type_metadata_ibfk_1',
            ),
            nullable=False,
            index=True,
        ),
        sa.Column('key', sa.String(255)),
        sa.Column('value', sa.String(255)),
        mysql_engine='InnoDB',
        mysql_charset='utf8',
    )


In API there will be two fields: "extra_specs" and "metadata" and in CLI those
fields will be "properties" and "metadata". The latter might confuse users, so
documentation is needed for both API and CLI to clarify, what to expect of each
field. The limit for this change is, that key-value pairs not meant for
scheduling purposes can still be put into the properties/extra_specs and will
lead to Errors in the scheduling process later on.

In a second phase metadefs can be added to standardize certain keys like
"encryption_type" or "replicated".

Alternatives
------------

We evaluated using the already existing user facing "extra_specs" field.
Administrators can already set key-value pairs here and user visible
extra_specs can be seen in this field. But this would lead to a problem in the
volume scheduler, as EVERY input in the extra_specs table will be evaluated by
the scheduler when looking for a fitting backend for the volume. While this
problem could be solved through a whitelisting or blacklisting approach,
maintaining and evaluating such lists is very opaque for users. Overall
this approach may lead to even more confusion in users.

Another option is to face use cases individually:

1. Information about whether a volume type creates encrypted volumes or not
could be calculated in the API calls for list/show volume types from the
presence of an encryption type. The information would be shown in the
"properties" field. This would be a very minimal patch, but will not solve
other use cases, that would benefit from user facing information.
2. Creating an extra field for the encryption in the volume type table, that
is automatically set when creating or deleting an encryption type. This
would need a database change and a change of the view of the volume_types.
3. Looking into the different drivers and how they handle internal
configurable replication and whether there are ways to let OpenStack know
this and propagate it to users. This is very hard to achieve, maybe even
impossible without input from an operator, who configured the backends and
volume types.
4. The policy for the encryption type could be loosend to let users see not
only whether a volume type has an encryption type, but also what algorithm
and provider is used for it. This may have a security impact.

All these options are not able to solve the general issue of having a
reliable way to provide user visible information in volume types. Neither will
they solve the problem, that operators are able to add key-value pairs not
meant for scheduling purposes to the volume type extra specs.


Data model impact
-----------------

It will be necessary to create a new table `metadata` that works like the
`extra_specs` table. As there are currently no user visible information in
volume types in place, an initially empty table would be sufficient.

To adhere the case of an upgrade from a previous OpenStack version, all volume
types need to be checked and for volume types with associated encryption types,
there need to be an encryption parameter set in the database according to the
chosen option. This should not be doable through an API call but will need
direct DB access, as changing the metadata of volume types in use should be
prohibited.


REST API impact
---------------

A new field needs to be added to the view of volume types. For that field new
API calls will be needed:

- a POST method to set new key-value pairs
- a DELETE method to delete a key-value pair.
- optionally a GET method to show a key-value pair.
- optionally add filtering for certain metadata for the GET volume types method

The volume type metadata will be handled like the volume type description, that
is, we will allow it to be modified even if the volume-type is in use. We can
do this because the volume type metadata is only descriptive, as opposed to the
volume-type extra_specs that affect the scheduler.

Additionally the create volume type API call should be able to handle key-value
pairs for the new metadata field.

Security impact
---------------

This change will expose a boolean value, that shows, whether a volume type
will encrypt volumes at creation or not.


Active/Active HA impact
-----------------------

There should not be any impact upon Cinder's Active/Active HA support.


Notifications impact
--------------------

There will be additional notifications when database entries are made or
removed. They will behave the same way as the notifications of extra_specs.
Those will be sent, whenever a key-value pair is created or deleted from the
new metadata field/database.


Other end user impact
---------------------

This change may lead to users wanting to use the filtering mechanism in
`openstack volume type list --metadata key=value` for both explained use
cases.


Performance Impact
------------------

The API call for getting volume types and their details will have an additional
query for the new metadata table.


Other deployer impact
---------------------

Old volume types with associated encryption types will need to have the
potentially new database entry set. This cannot be done via `volume type set`
command, but has to be written into the database either manually or with a
script, that sets the correct entry for all affected volume types.


Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Josephine Seifert (josei)

Other contributors:
  None

Work Items
----------

* Add a "volume_type_metadata" table to the Cinder database

* Add new API-Endpoints for the metadata field of volume types and add new
  possibility to create new key-value pairs in the metadata field when
  creating a volume type

* Add support in CLI/SDK for the new API calls


Dependencies
============

There are no dependencies.

Testing
=======

The test case for creating a volume type may be updated to integrate the
addition of a metadata key-value pair.

Documentation Impact
====================

There will be a documentation needed to separate between the extra specs
and the handling of user visible information.


References
==========

* `Mailing list discussion <https://lists.openstack.org/archives/list/openstack-discuss@lists.openstack.org/thread/PP7IMXVOO2SM47JRMDYYVB2IA3XIEZD5/>`_

* `Discussion in the Cinder Midcycle <https://etherpad.opendev.org/p/cinder-caracal-midcycles#L93>`_

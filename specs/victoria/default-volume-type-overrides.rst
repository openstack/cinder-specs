..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Default volume type overrides
=============================

https://blueprints.launchpad.net/cinder/+spec/multiple-default-volume-types

Cinder's current default volume type options are limited and are insufficient
for big and/or complex deployments.

This spec proposes adding per project volume types.

Problem description
===================

On complex and/or big OpenStack deployments we may have multiple availability
zones (AZs) with many projects, increasing the complexity of properly
structuring and managing resources.

One of the problems we encounter when having multiple AZs is that we usually
want our projects to have volumes and instances collocated within the same AZ,
but we can only define one default volume type, forcing our users to always
remember to create volumes with a volume type (if it has the AZ defined in it)
or to explicitly pass the AZ.

This is error prone, and users forgetting it will lead to unwanted cross AZ
attachments, with the consequent performance issues and network throughput
costs.

Right now there are 2 workarounds that users are adopting to mitigate this
situation:

- Disabling cross AZ attachments in Nova: For deployments that don't want to do
  cross AZ instance live migrations, they can disable cross AZ attachments in
  Nova so the attachment to the instance fails, thus preventing performance
  issues on running VMs.  This solution usually forces users to recreate the
  volume in the right AZ.

- Unusable default volume type: Setting a default volume type with a self
  explanatory name, such as *Need_To_Select_Volume_Type* and setting it in a
  way that will fail the scheduling and leave the volume in an error state.
  This is less wasteful, but still not a nice user experience.

This spec addresses this specific problem.

Use Cases
=========

1. On a country wide OpenStack deployment that spreads across multiple data
   centers where each one has its own storage array and OpenStack project, an
   administrator wants to make sure that volumes are created in their own data
   center by default.  For this purpose the admin defines a per project default
   volume type where it has the appropriate extra specs.

2. On an OpenStack deployment with 3 different volume types (high performance,
   replicated, compressed) and different department (each with a different
   project) have different requirements, so the system admin wants to assign a
   different OpenStack volume type to each one of them.


Proposed change
===============

The proposal is to add the possibility to override the existing Cinder default
Volume Type on a per project basis to make it easier to manage complex
deployments.

With the introduction of this new default volume type configuration, we'll now
have 2 different default volume types.  From more specific to more generic
these are:

- Per project
- Defined in cinder.conf (defaults to *__DEFAULT__* type)

So when a user creates a new volume that has no defined volume type (explicit
or in the source), Cinder will look for the appropriate default first by
checking if there's one defined in the DB for the specific project and use it,
if there isn't one, it will continue like it does today, using the default type
from ``cinder.conf``.

Administrators and users must still be careful with the normal Cinder behavior
when creating volumes, as Cinder will still only resort to using the default
volume type if the user doesn't select one on the request or if there's no
volume type in the source, which means that Cinder **will not** use any of
those defaults if we:

- Create a volume providing a volume type
- Create a volume from a snapshot
- Clone a volume
- Create a volume from an image that has ``cinder_img_volume_type`` defined in
  its metadata.

By default the policy restricting access to set, unset, get or list all
project default volume type will be set to system admins only.

Alternatives
------------

An alternative would be allow setting default volume types overrides on a per
user and per project basis, as well as globally.

`Such alternative was discarded
<https://review.opendev.org/#/c/733555/2/specs/victoria/default-volume-type-overrides.rst>`_,
given the added complexity and effort needed to implement to provide very
little benefits, as configurable per user and global default volume types don't
seem to be a user necessity.

Data model impact
-----------------

We will add a new table called ``default_volume_type`` that will have 3 fields:

* ``id`` primary integer key.
* ``volume_type_id`` foreign key string of length 36.
* ``project_id`` string.

We won't reuse existing ``volume_type_projects`` table and just add an
``is_default`` field because then the combinations of private/public volume
types and defaults get complex as we update or remove them.

In terms of database migrations we'll just have to create the table.

REST API impact
---------------

We'll need a new set of REST API calls to provide the CRUD operations:

* Set the default volume type (create or update) for a specific project.

  Besides ensuring that the volume type exists and the project has access to
  it, this method will also call keystone to validate that the project exists.

  * PUT /v3/default-types/{project_id}

    * JSON body to set default type (by name or id) for a project:

      .. code-block:: json

         {
           "volume_type": "lvm"
         }

    * Response Codes:
        - Success - 200 (with body)

          .. code-block:: json

             {
               "project_id": "248592b4-a6da-4c4c-abe0-9d8dbe0b74b4",
               "type_id": "f8a82360-0b1a-4649-8615-114341dd06e0"
             }

        - Error - 400 (volume type not found), 404 (project id not found)

* Unset a default volume type.  It will fail if there is not a default
  volume type for the given project or if the project no longer exists in
  keystone.

  * DELETE /v3/default-types/{project_id}

      * Response Codes:
        - Success - 204
        - Error - 404 (project id not found)

* Get volume types

  * Show default type for a project
    GET /v3/default-types/{project_id}

    * A request with project_id will return a single JSON object containing
      the project_id and type_id of the specified project.  Returns 404 error
      code if the entry is not in the DB, independent of the existence of the
      project in keystone.

      .. code-block:: json

         {
           "project_id": "248592b4-a6da-4c4c-abe0-9d8dbe0b74b4",
           "type_id": "f8a82360-0b1a-4649-8615-114341dd06e0"
         }

    * Response Codes:
      - Success - 200 (with body)
      - Error - 404 (project id not found or no default type)

  * List all default types
    GET /v3/default-types

    * A request without a project_id will return all the volume type
      defaults defined.  A sample response would be:

      .. code-block:: json

         [
             {
               "project_id": "248592b4-a6da-4c4c-abe0-9d8dbe0b74b4",
               "type_id": "f8a82360-0b1a-4649-8615-114341dd06e0"
             },
             {
               "project_id": "1234567-4c4c-abcd-abe0-1a2b3c4d5e6ff",
               "type_id": "5c4df055-571e-4430-9823-416b82f337b2"
             }
         ]

    * Response Codes:
      - Success - 200 (with body)

      Notice that we only list overrides, we won't return the value of
      ``default_volume_type``.

A user can get its effective default type using existing ``cinder
type-default`` command: ``GET /v3/{project_id}/types/default``.

Security impact
---------------

None

Active/Active HA impact
-----------------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

There will be a new set of commands in the *python-cinderclient* to match the
new REST API endpoints:

* Set default: ``cinder default-type-set <type-name> <project-id>``

* Unset default: ``cinder default-type-unset <project-id>``

* List defaults: ``cinder default-type-list [--project <project-id>]``

Performance Impact
------------------

Create volume operations that don't define a default volume type (explicitly or
via a source) will have a tiny performance impact since we'll add a DB query to
get the defaults.

Other deployer impact
---------------------

None

Developer impact
----------------

We should no longer refer directly to the ``default_volume_type`` configuration
option throughout the code and instead use the ``get_default_volume_type``
method from ``cinder.volume.volume_types``.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <whoami-rajat>

Work Items
----------

* Cinder service

  * Check if caller is authorized to do the operation: First we'll check the
    normal policy to see if it's a system admin, etc, but then we'll have to
    check the project, and we'll only authorize the operation if caller's
    context has system scope.

    For this we have introduced a new policy to check if the caller is a
    system admin and then leverage the `get_project_hierarchy` method in
    `cinder.quota_utils` to validate that the project actually exists
    (since the method does a `get` of the project).

  * Add the DB field and the DB migration.

  * Add 3 DB layer methods:

    * Set default volume type: Given a volume type id and a project id this
      method sets its default volume type.

      It will try to update the ``volume_type_id`` for the ``project_id`` and
      if it fails because the row doesn't exist, then it will create the DB
      row.

    * Unset default volume type: This will set ``deleted`` and ``deleted_at``
      fields for the ``project_id``.  If it fails because it doesn't exist, the
      failure will not be propagated and it will be considered a success.

    * Get projects default volume type: Returns a list of project and volume
      type ids limited by the ``project_id`` if it is provided.

  * Update ``get_default_volume_type`` to return the effective volume type for
    the current project.  Basically calling the *get project default type* DB
    method, and if it returns None, then we'll continue with the current code
    we have to use the one from the config.

  * Updating the volume type methods to ensure we don't try to delete a volume
    type that is used as a default, and making sure we don't set as private a
    volume type that a project is using as a default, and such operations.

  * Ensure that ``purge_deleted_rows`` from ``cinder.db.sqlalchemy.api`` works
    as expected.

  * Add a new API microversion and implement the 4 REST API methods.

  * Write appropriate unit-tests for the DB methods, REST API methods, and
    update existing tests for the changes we introduced.

* Cinder client: Add the 3 commands mentioned earlier in the `Other end user
  impact`_ section.

* Tempest tests: Add tempest tests as describe in the `Testing`_ section.

* Documentation as describe in the `Documentation impact`_ section.

Dependencies
============

None

Testing
=======

Besides writing the appropriate unit-tests for the DB methods, REST API
methods, and update existing tests for the changes we introduced, we also need
a series of Tempest tests to test existing functionality.

* Confirm that the priority of the default types are observed:

  * Admin creates a custom volume type and sets it as the project's default
  * Create an empty volume with a normal user and check that the volume type is
    the one we created, then delete it.
  * Create another volume with the admin user and see that it works the same.
  * Create an empty volume with the alternative project admin user and confirm
    it doesn't use our custom volume type.
  * Unset the custom volume type we set in step 1.
  * Create an empty volume and check that the volume type is not the custom
    one.

* Confirm that ``cinder type-default`` works as expected:

  * Admin confirms that there are no default overrides for the project and
    alternative project.
  * Get current default using ``type-default``.
  * Create 2 custom volume types: #1 and #2
  * Set default volume type #1 for project and #2 for alternative project.
  * Request the type default with both projects and confirm we get the ones we
    have just set.
  * Unset the custom volume types.
  * Get current default using ``type-default`` and confirm is the same as in
    step 2.

* Confirm that listing/showing default type overrides works as expected:

  * Admin confirms that there are no default overrides for the project by first
    listing the default-types, and then by showing the default type for a project
    (we'll get 404), for both the normal project and the alternative.
  * Create 2 custom volume types: #1 and #2
  * Set default volume type #1 for project and #2 for alternative project.
  * Admin lists all default volume type and validates them.
  * Admin gets default volume type for project and confirms that it only gets
    that one.
  * Repeat previous 2 steps for the alternative project.
  * Unset the default types.
  * Confirm that default type list returns empty list.
  * Confirm that showing default for a project id returns 404.
  * Show default for a fake project id and confirm we get 404 error code.

Documentation Impact
====================

A description of the default volume types overrides behavior will be added to
the admin section, for example on path
``doc/source/admin/default-volume-types.rst``.

This file will be linked from ``doc/source/admin/index.rst`` and
``doc/source/configuration/index.rst`` since it is part of the configuration
but it's also an administrative task.

The new CLI commands will be listed and explained in
``doc/source/cli/cli-manage-volumes.rst`` with examples of the new CLI
commands.

Additionally the new REST API calls will need to be documented in
``api-ref/source/v3/default-volume-types.inc`` and samples added to
``api-ref/source/v3/samples``.

And the microversion history will need to be updated in
``cinder/api/openstack/rest_api_version_history.rst``.

References
==========

* `Untyped volume to default volume type
  <https://specs.openstack.org/openstack/cinder-specs/specs/train/untyped-volumes-to-default-volume-type.html>`_

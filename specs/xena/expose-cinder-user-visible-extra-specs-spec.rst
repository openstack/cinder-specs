..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Expose user-visible extra specs in volume types
=================================================

https://blueprints.launchpad.net/cinder/+spec/expose-user-visible-extra-specs

This spec proposes definition of a class of *user visible* extra specs.
These will be shown to regular users (members or readers in a keystone
project) when they list extra specs or show volume types or extra specs.

Problem description
-------------------

Cinder only allows cloud administrators to see the extra specs in volume
types. This makes sense for specs that control knobs for use by drivers,
that select a particular backend, or that would otherwise reveal backend
specifics. But it is problematic for backend independent extra specs
like ``multiattach`` or ``RESKEY:availability_zones`` – specs that
carry critical information for regular users when they select volume
types to use as they create volumes.

The traditional response to this problem has been that cloud
administrators should add this critical information to the volume-type
``description`` field or provide it in the user documentation for their
cloud.

Whatever one thinks of the adequacy of that response when the cloud
users are human beings, it clearly fails when the end user is an
automated agent.

Use Cases
---------

OpenShift cluster storage operators are automated agents that set up
Kubernetes Storage Classes on underlying infrastructure. When that
infrastructure is provided by OpenStack, the Cinder CSI operator will
create a Storage Class corresponding to each Cinder volume type.
OpenShift users in turn reference these Storage Classes in Persistent
Volume Claims when dynamically provisioning Persistent Volumes for use
by containerized applications.

When provisioning a persistent volume via Cinder CSI with ReadWriteMany
access mode, the persistent volume claim should specify ``block``
volumeMode (default is ``filesystem``) and reference a Storage Class
corresponding to a Cinder volume type with a ``"multiattach": "<is> True"``
extra spec.

Because OpenShift administrators and all the software that runs on their
behalf are in the general case just regular OpenStack users rather
than OpenStack administrators, they currently lack the ability to
discover extra specs for volume types corresponding to their Storage
Classes and are hindered in their ability to “do the right thing” for
OpenShift users.

Cinder CSI now has a ``topology`` feature as well where it will likely
make sense to be able to see ``RESKEY:availability_zones`` extra specs when
setting up Storage Classes corresponding to volume types.

Proposed change
---------------

Define a set of ``user visible`` extra specs in the Cinder code and modify the
REST API view for showing extra specs to reveal these in policy based
request contexts. The initial set of ``user visible`` extra specs is
sufficient to establish a framework, and meet the immediate needs described in
the use cases. The set is intentionally small, and leverages existing extra
specs. The following extra spec keys are to be treated as ``user visible``:

- ``multiattach``
- ``RESKEY:availability_zones``
- ``replication_enabled``

The set of ``user visible`` extra specs will be a fixed list defined in the
Cinder code, and can be extended but only by enhancing the code. Future
updates can support additional volume characteristics that can be expressed with
an extra spec. For example, it might be useful for users to know whether a
volume type provides volumes that support online extend, but there currently
is no extra spec associated with that feature.

The REST API behavior will be policy based, and not require a new
microversion. The existing policies for accessing extra specs will remain, but
the default values will be changed to grant access to any authorized user. A
new policy will determine whether access is granted to all extra specs, or
just ``user visible`` extra specs.  The policies will allow users to view
extra specs, but the view will be restricted to the extra specs that are
considered ``user visible``. A new policy will control access to all extra
specs, including the ones that are ``user visible``.

The current policies (and their default values) for accessing extra specs are:

========================================= ==========
Policy                                    Default
========================================= ==========
volume_extension:access_types_extra_specs admin-only
volume_extension:types_extra_specs:index  admin-only
volume_extension:types_extra_specs:show   admin-only
========================================= ==========

The proposed policies, and their new default values, will be:

================================================= ==========
Policy                                            Default
================================================= ==========
volume_extension:access_types_extra_specs         any authorized user
volume_extension:types_extra_specs:index          any authorized user
volume_extension:types_extra_specs:show           any authorized user
volume_extension:types_extra_specs:read_sensitive admin-only
================================================= ==========

The new ``volume_extension:types_extra_specs:read_sensitive`` policy will
govern whether access is granted to all extra specs, or is limited to just
``user visible`` extra specs. Hereafter in this spec, the new policy will
be abbreviated as ``read_sensitive``.

Alternatives
------------

Build a new API to return user visible capabilities and features from
volume types without exposing them as “extra specs.” Although this
alternative would also solve the problem, it requires more code to
write, test, and document.

Control exposure of each individual extra spec entirely by policy. It is
not clear how to do this without significant changes to the existing
REST API. Nor in our opinion is it desirable. There are clear cut
examples of backend independent capabilities and features that are
properly the business of any user who is authorized to create volumes, so
users should have a reasonable expectation that these can be discovered
without burdening the cloud administrator with the task of constructing
policies for them one by one.

The new ``volume_extension:types_extra_specs:read_sensitive`` policy is
introduced, but the existing policies remain defaulting to admin-only. The
downside is this would require cloud administrators to modify the default
policies for the ``user visible`` extra specs to be available to non-
administrative API requests. However, the entire premise is that ``user
visible`` extra specs are, by definition, intended to be visible to users.
Cloud administrators shouldn't need to opt-in to reap the benefits of this
feature. Cloud administrators who wish to retain the current behavior, where
users are not authorized to see any extra specs, may do so by explicitly
setting these policies to admin-only (the previous default value).

- volume_extension:access_types_extra_specs
- volume_extension:types_extra_specs:index
- volume_extension:types_extra_specs:show

Data model impact
-----------------

None

REST API impact
---------------

The REST API affected by this spec are:

#. /v3/{project_id}/types
#. /v3/{project_id}/types/{volume_type_id}
#. /v3/{project_id}/types/{volume_type_id}/extra_specs
#. /v3/{project_id}/types/{volume_type_id}/extra_specs/{key}

The following examples document the behavior for a volume type with two
extra specs:

- ``multiattach`` (a ``user visible`` extra spec)
- ``volume_backend_name`` (which will be visible only when the context
  satisfies the ``read_sensitive`` policy)

GET /v3/{project_id}/types
~~~~~~~~~~~~~~~~~~~~~~~~~~

This REST API returns a list of volume types, and the following example
shows just a single volume type.
When the context doesn't satisfy the ``read_sensitive`` policy, the
``extra_specs`` field is included but only ``user visible`` entries are
present.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:access_types_extra_specs         True
volume_extension:types_extra_specs:read_sensitive False
================================================= =======

.. code-block:: python

  {
      "volume_types": [
          {
              "id": "6685584b-1eac-4da6-b5c3-555430cf68ff",
              "qos_specs_id": null,
              "name": "vol-type-001",
              "description": "volume type 0001",
              "os-volume-type-access:is_public": true,
              "is_public": true,
              "extra_specs": {
                  "multiattach": "<is> True"
              }
          }
      ]
  }

When the ``read_sensitive`` policy is satisfied (by default, this will be the
case only for administrators) the full list of ``extra_specs`` is returned.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:access_types_extra_specs         True
volume_extension:types_extra_specs:read_sensitive True
================================================= =======

.. code-block:: python

  {
      "volume_types": [
          {
              "id": "6685584b-1eac-4da6-b5c3-555430cf68ff",
              "qos_specs_id": null,
              "name": "vol-type-001",
              "description": "volume type 0001",
              "os-volume-type-access:is_public": true,
              "is_public": true,
              "extra_specs": {
                  "multiattach": "<is> True",
                  "volume_backend_name": "SecretName"
              }
          }
      ]
  }

GET /v3/{project_id}/types/{volume_type_id}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The REST API shows the details for the specified volume type, and the behavior
is similar to the to the /v3/{project_id}/types API.

- The ``extra_specs`` field will contain any ``user visible`` extra specs.
- When the ``read_sensitive`` policy is satisfied, the full list of
  ``extra_specs`` is returned.

GET /v3/{project_id}/types/{volume_type_id}/extra_specs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The policy will be updated to allow any authorized user access to this
API. But, similar to above, for non ``read_sensitive`` requests the list of
``extra_specs`` will be limited to those that are ``user visible``.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:types_extra_specs:index          True
volume_extension:types_extra_specs:read_sensitive False
================================================= =======

.. code-block:: python

  {
      "extra_specs": {
          "multiattach": "<is> True"
      }
  }

If there are no ``user visible`` extra specs defined for the specified
volume type, then an empty dictionary will be returned.

.. code-block:: python

  {
      "extra_specs": {}
  }

When made by an administrator, the response will be the complete set of extra
specs (no change from current behavior).

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:types_extra_specs:index          True
volume_extension:types_extra_specs:read_sensitive True
================================================= =======

.. code-block:: python

  {
      "extra_specs": {
          "multiattach": "<is> True",
          "volume_backend_name": "SecretName"
      }
  }

GET /v3/{project_id}/types/{volume_type_id}/extra_specs/{key}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When the specified extra-specs {key} is in the ``user visible`` set, the value
will be returned regardless of whether the requester satisfies the
``read_sensitive`` policy.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:types_extra_specs:show           True
volume_extension:types_extra_specs:read_sensitive N/A
================================================= =======

.. code-block:: python

  {
      "multiattach": "<is> True"
  }

The response to requests for extra specs that are not ``user visible`` will
depend on the context. If the request satisfies the ``read_sensitive`` policy
then the appropriate response is returned.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:types_extra_specs:show           True
volume_extension:types_extra_specs:read_sensitive True
================================================= =======

.. code-block:: python

  {
      "volume_backend_name": "SecretName"
  }

But if the request does not satisfy the ``read_sensitive`` policy then the
API will return 404 NOTFOUND. Returning 404 (instead of 403) prevents leakage
of the names of ``read_sensitive`` extra spec keys.

================================================= =======
Policy                                            Context
================================================= =======
volume_extension:types_extra_specs:show           True
volume_extension:types_extra_specs:read_sensitive False
================================================= =======


.. code-block:: python

  {
      "itemNotFound": {
          "code": 404
          "message": "Volume Type 6685584b-1eac-4da6-b5c3-555430cf68ff has no extra specs with key volume_backend_name."
      }
  }

Security impact
---------------

None. The set of ``user visible`` extra specs is hard coded and does not
include extra specs that reveal back end details.

Active/Active HA impact
-----------------------

None. REST API layer change with no locking or exclusion or service
restart implications.

Notifications impact
--------------------

None – no asynchronous operations involved.

Other end user impact
---------------------

Cinderclient and OSC and Horizon should not need changes. Regular
users will see some new information but no new fields are required.

Performance Impact
------------------

None

Other deployer impact
---------------------

No impact on OpenStack deployment tools or deployment planning/design.

Developer impact
----------------

No impact on OpenStack development outside the scope of this change.

Implementation
--------------

Assignee(s)
~~~~~~~~~~~

Primary assignee: abishop

Work Items
~~~~~~~~~~

-  Define list of user visible extra specs.
-  Define the ``read_sensitive`` policy, and modify the existing default
   policy rules that govern access to extra specs to allow access to any
   authorized user.
-  Update the API responses per this spec.
-  Modify unit tests to check for exposure of exactly and only the
   user visible extra specs without ``read_sensitive`` authorization.
-  Tempest tests

Dependencies
~~~~~~~~~~~~

Testing
~~~~~~~

-  New tempest tests in cinder-tempest-plugin

Documentation Impact
~~~~~~~~~~~~~~~~~~~~

-  Add section to Cinder Administration that defines and lists user
   visible extra specs. Describe the new ``read_sensitive`` policy, and
   explain the policy settings that will give an operator "legacy" behavior.
-  Update API reference

References
----------

None

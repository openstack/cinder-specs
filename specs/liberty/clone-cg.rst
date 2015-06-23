..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Clone Consistency Group
==============================

https://blueprints.launchpad.net/cinder/+spec/clone-cg

This proposal is to add a feature to create a cloned consistency group
from an existing one.

Problem description
===================

Currently a user can create a Consistency Group first and then clone
one volume from the existing CG at a time and add it to the new CG
until all volumes are cloned and added to the CG.

The proposal wants to enhance the existing create CG from source API
and make this a one step rather than multi-step process.

Note: The cloned volumes in the new CG are not guanranteed to be
consistent. From the consistency point of view, it's no different from
creating a empty CG first and then creating volume one after another and
adding to the CG.

The existing Create CG from source API allows user to create a CG from
the group snapshot (CG snapshot), which should be consistent.

Use Cases
=========

Suppose a user already has a CG with volumes in it. He/she now wants
to create another CG with all volumes cloned from the other CG. The
proposed change will make this much easier for the user.

Proposed change
===============

* The existing Create CG from Source API takes an existing CG snapshot
  as the source.
* This blueprint proposes to modify the existing API to accept an existing
  CG as the source.

Alternatives
------------

Without the proposed changes, we can create a CG from an existing CG
with these steps:
* Create an empty CG.
* Create a cloned volume from an existing volume in an existing CG
  and add to the new CG.
* Repeat the above step for all volumes in the CG.

Data model impact
-----------------

* DB Schema Change: A new column source_cg_id will be added to the
  consistencygroups table.

REST API impact
---------------

Consistency Group API change

* Create Consistency Group from Source
  * V2/<tenant id>/consistencygroups/create_from_src
  * Method: POST
  * JSON schema definition for V2::

        {
            "consistencygroup-from-src":
            {
                "name": "my_cg",  # existing
                "description": "My consistency group",  # existing
                "cgsnapshot_id": "xxxxxxxx",  # existing
                "consistencygroup_id": "xxxxxxxx",  # new
            }
        }

  * In the existing Create Consistency Group from Source API, add a new
    parameter "consistencygroup_id" to allow an existing CG to be the source
    of the new CG.

* Cinder Volume Driver API
  Two new optional parameters will be added to the existing volume driver API:
    * def create_consistencygroup_from_src(self, context, group, volumes,
      cgsnapshot=None, snapshots=None, src_group=None, src_volumes=None)
    * Note only "src_group" and "src_volumes" are new parameters.

Security impact
---------------


Notifications impact
--------------------


Other end user impact
---------------------

python-cinderclient needs to be changed to support the modified API.

* Create Cloned CG
  cinder consisgroup-create-from-src --name <name> --description <description>
  --consistencygroup <cg uuid or name>

Performance Impact
------------------


Other deployer impact
---------------------

None.

Developer impact
----------------

Driver developers can add support to this feature in the modified driver API.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xing-yang

Other contributors:

Work Items
----------

1. API change:
   * Modify Create CG from Source API
2. Volume Driver API change:
   * Modify corresponding driver API
3. DB schema change

Dependencies
============

Testing
=======

New unit tests will be added to test the changed code.

Documentation Impact
====================

Documentation changes are needed.

References
==========

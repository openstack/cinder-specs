..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Transfer snapshots with volumes
==========================================

https://blueprints.launchpad.net/cinder/+spec/transfer-snps-with-vols

This spec aims to extend transfer function that can transfer snapshots
with volumes at same time.

Problem description
===================

Currently, Cinder can transfer volumes to another project's owner, but
if this volume has snapshots before transferring, after this operation,
new owner can't delete this volume successfully since it has snapshots.

Use Cases
=========

When transferring volume with snapshots, a user will now have a choice whether
or not to transfer those snapshots along with a volume. By default, snapshots
are transferred, but selecting this option will allow snapshots not to be
transferred to another project.
Note those snapshots can be deleted by the remote user if necessary.

Proposed change
===============

In normal deleting, Cinder will disallow the request since the volume has
snapshots,  but unfortunately, Cinder has cascade deleting operation now,
the request will be passed down to driver, some driver will raise
exception since snapshot is still existing. So it also should be changed in
cascade deletion process.

* Add an optional argument "--no-snapshots" in transfer API and CLI. If user
  didn't specify it, cinder will transfer snapshots that a volume has by
  default.
* Add a new field in transfer DB model to record this option.
* Update snapshot's information in DB like user id, project id, etc.
* Check if the volume still has some snapshots in other project when cascade
  deleting.


Alternatives
------------

Another option is cinder transfer the snapshots if user specifies an option
argument '--with-snapshots'.

This option can be minimal with the change, in order to make the client code
more simple.

Data model impact
-----------------

Add a new field "with_snapshots(boolean)" in transfer model.


REST API impact
---------------

*New microversion in Cinder API.

*Add an optional argumet "no_snapshots"::

  POST /v3/{project_id}/os-volume-transfer
  RESP BODY: {"transfer": {
                           ...
                           no_snapshots: [True/False],
                          }
             }


Security impact
---------------

If users didn't transfer snapshots with volume, there could be kind of
security impact that remote users may be able to act upon the untransferred
snapshots. For example, by leveraging COW, that remote user can change the
snapshot size in backend.

Notifications impact
--------------------

Add 'with_snapshots' information to transfer notificaiton.

Other end user impact
---------------------

None

Performance Impact
------------------

There could be a db performance issue if a lot of snapshots associated with a
given volume since cinder need to change those snapshots' project id, user id,
etc.

Other deployer impact
---------------------

None


Developer impact
----------------

Drivers that implement some form of volume ownership tracking on
the backend will need to be fixed to track this change too.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  wanghao<sxmatch1986@gmail.com>

Other contributors:
  None

Work Items
----------

* Implement feature in Cinder tree.
* Update cinderclient to support this functionality.
* Add change to API doc.


Dependencies
============

None


Testing
=======

Both unit and Tempest tests are needed to be created to cover the code change.


Documentation Impact
====================

The cinder API documention will need to be updated to reflect the REST
API changes.

Devref entry on the volume transfer driver entry point should be created.


References
==========
None

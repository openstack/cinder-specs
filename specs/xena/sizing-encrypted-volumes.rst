..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Sizing encrypted volumes
========================

https://blueprints.launchpad.net/cinder/+spec/sizing-encrypted-volumes

Problem description
===================

If you have an unencrypted volume and migrate to an encrypted type, it fails
because there isn't enough space on the target to fit the entire source
volume. This happens because an encrypted volume must have a header, which
takes up some space.

In fact, the exact amount of space lost isn't clear, it may vary by
LUKSv1/LUKSv2 and PLAIN dm-crypt formats. It's probably on the order of 1-2
MB, but we allocate space in GB. Whether you can do really fine-grained space
allocation or not probably depends on the backend containing the volume.
However, we also need to keep the gigabyte boundaries for accounting purposes
(usage, quotas, charges, etc.) because that's what everyone's tooling expects.

In addition, drivers optimize migration in different ways, and since we
previously didn't have a new size parameter, there's no easy way to get this
info into the drivers.

Taking everything into consideration, we could have a flag indicating that
it's OK for the volume to be increased in size if it's required for the
retype to succeed. This way we don't go behind the user's back -- we fail if
the volume won't fit, and allow them to retype to a new size if they want the
retype to succeed.

Use Cases
=========

The main use-case here is when upgrading an old deployment and would like to
encrypt old volumes after that.

Proposed change
===============

* The manager.py will create a destination volume bigger <source-volume-size+1>
  GB size if the flag ‘allow-resize==on_demand’ and the destination volume-type
  is encrypted.

* Driver-specific method ‘_copy_volume_data’ for copy volume data when using block
  devices (i.e LVM will still use the actual _migrate_volume_generic). This method
  will be called after _before_volume_copy during volume migration.

* RPC minor version for a new param
  3.x - Add allow_resize to retype method and increase_size to migrate method.

* Add a new optional field, "allow_resize", to the os-retype action. Possible
  values will be "on-demand" or "never" (which are the same as the current
  "migration_policy" field).  If not specified, the default is "never".

* Add a new optional field, "increase_size", to the os-migrate action. The
  operator specifies how many GB to grow the volume (so the final size is current
  size + increase_size).

* Add the next parameters to Python-cinderclient for retype and migrate commands:

  - Retype: New flag ``--allow-resize`` indicating that it's OK for the volume to be
    increased in size if it's required for the retype to succeed.

    .. code-block:: bash

      Optional Arguments:
      --allow-resize <never|on-demand>
      Allow the volume to be increased in size during the retype if the system
      decides it is necessary. This argument is recommended when retyping to
      an encryption type.
      ...

  - Migration: allow a new size to be specified during a volume migration. New
    optional argument added ``--increase-size'``

    .. code-block:: bash

     Optional Arguments:
     --increase-size [<size>] Number of GiBs to grow the volume during migration.
     This is recommended when migrating to an encrypted host.
     ...

* The displayed size of the volume will be the actual size of the volume.

Alternatives
------------

Leave everything as it is and don’t allow this kind of migration. To help users
add some workaround to the documentation (i.e. backup the volume and then restore
the volume to a bigger one). <1GB size increases won't be allowed, as explained
in the problem description above.

Data model impact
-----------------

None

REST API impact
---------------

* Microversion bump

* New parameters should be passed to retype and migrate commands as explained in
  the ‘proposed change’ section.

Security impact
---------------

None. If you're allowed to migrate at all, you should be able to increase the size,
and if you're allowed to retype, you should be able to allow-resize.

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

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sofia Enriquez <lsofia.enriquez@gmail.com>

Work Items
----------

* Implement new logic in cinder
* Implement new logic in python-cinderclient
* Unit-tests
* Tempest tests

Dependencies
============

None


Testing
=======

Unit tests and current devstack-based jobs won’t be enough to test these
changes. At least two tests should be added to
cinder_tempest_plugin/api/volume/admin/test_volume_retype.py

1. retype unencrypted volume to LUKS type.
2. retype encrypted LUKS volume to regular type.

Documentation Impact
====================

Currently I’ve added some documentation notes warning about retyping an
unencrypted/encrypted volume. However, these notes should be replaced with
proper documentation.

Notes to be removed
- https://review.opendev.org/#/c/732988/
- https://review.opendev.org/#/c/745199/


References
==========

* https://etherpad.opendev.org/p/sizing-encrypted-volumes
* Victoria Mid Cycle https://wiki.openstack.org/wiki/CinderVictoriaMidCycleSummary
* Victoria PTG https://wiki.openstack.org/wiki/CinderVictoriaPTGSummary#Sizing_encrypted_volumes

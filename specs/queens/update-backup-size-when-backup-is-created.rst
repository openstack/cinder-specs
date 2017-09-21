..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Update backup's size when backup is created
===========================================

https://blueprints.launchpad.net/cinder/+spec/report-size-when-backup-created

This blueprint proposes to update backup with the size reported by
backup backend when backup is created.

Problem description
===================

Most clouds provide end user a convenient way to protect their data by
creating a backup for their volume. Users can create any backup copy at
anytime, and the cloud providers charge them by the backup size in total.
As the basic volume service, Cinder supports creating, restoring and
deleting backups plus the backup quotas, but our backup size isn't correct
at present, the backup's size always equals to the original volume's size
even if the incremental backup doesn't have any content in it.

Use Cases
=========

Backup's size attribute will reflect the actual size on the backend, the quota
and charge would be more accurate.

Proposed change
===============

This spec proposes that the backup driver should report the actual size that
object holds at the backend when backup is created. So, when user requests to
create a backup for a volume with 10G, Cinder will still generate the record
with 10G as well as consume 10G for quota reservation, but when the actual
size is reported, we will update both the object and the quotas.

**Note**: The actual size here stands for the final size at the backend, that
means if we create backup for a 1G volume which only has 200mb data and
after compression only 100mb is used at disk, the actual size is 100mb,
not 200mb or 1G.

As Cinder has the unified unit G for every resource, that size should be
rounded up to G too, for example, if we create an incremental backup whose
actual size is only 12mb at backend, driver should still report 1 G to
Cinder service:

.. code-block:: python

    def backup(self, backup, volume_file, backup_metadata=False):
        """Start a backup of a specified volume.

        Driver should return the size that the backup object holds
        in the backend, with the unit of G. For instance:

        .. code-block:: python

            {
                'size': 2
            }
        """
        return

Cinder's backup service would use this ``size`` to update backup model
as well as commit portion of the reservation (Instead of commit the
reservation record, Cinder will update the reservation and then commit
the updated record after backup is actually created in the background
to cover this case ).

In order to report the actual size that backup holds, drivers need
to record or calculate the object size. Take our ``chunkeddriver`` for
instance, the total size is accumulated by the objects' size::

    size = object1_size + object2_size....

If compression feature is enabled, the object's size should be::

    size = compressed_object1_size + compressed_object2_size...

Now Cinder has the attribute ``size`` for backup object which
only reflects the volume's size when created. In order to support
updating the actual size of the backup, we will add one more attribute
here.

1. **volume_size**: This is used to record the original volume's size
and can be used to create new volume when restoring, we can not directly
link to the original volume size here because we could resize the volume
after the volume is backed up.

2. **size**: This will be used to store the reported value from driver
rather the original volume size. And when backup is first created,
this value would be equal to the ``volume_size`` and will be updated
when backup is created at backend.


Alternatives
------------

Keep using volume's size to generate the backup's size as well as quota
usage record in database.

Data model impact
-----------------

None

REST API impact
---------------

None

Cinder-client impact
--------------------

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

There would be a slight performance impact
when committing quota reservations.

Other deployer impact
---------------------

None

Developer impact
----------------

Developer should report backup's size in ``backup`` method
when adding new backup driver.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tommylikehu(tommylikehu@gmail.com)

Work Items
----------

* Update cinder to support update the backup size.
* Update existing driver to report backup's actual size.
* Add related unit testcases.

Dependencies
============

None

Testing
=======

* Add unit tests to cover this change.

Documentation Impact
====================

* Update the base backup driver's interface.
* Update developer's documentation to advertise this change.

References
==========

None

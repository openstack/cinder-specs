..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


========================================
Support for incremental backup in Cinder
========================================
Launchpad Blueprint:
  https://blueprints.launchpad.net/cinder/+spec/incremental-backup

Problem description
====================
The current implementation of Cinder Backup functionality only supports full
backup and restore of a given volume. There is no provision to backup changes
only that happened since last backup. As the volumes grow bigger and over,
all changes to volumes between backups stay relatively small, copying
entire volumes during backups will be resource intensive and do not scale well
for larger deployments. This specification discusses implementation of
incremental backup feature in detail.

Use Cases
=========

Proposed change
================
Cinder backup API, by default uses Swift as its backend. When a volume is
backed up to Swift, Swift creates a manifest file that describes the contents
of the backup volume. The manifest file contains header (metadata) and array
of pointers to the volume backup files. Since Swift has an upper limit on the
object size, Cinder backup API splits the volume data into individual chunks
of Swift object size and uploads these individual chunks to Swift. Cinder
volume backup manifest file includes these list of objects, their corresponding
objects, the logical offset of each object within the volume and a message
digest of each chunk to detect any unwarranted changes to objects. During
restore operation, Cinder reconstructs the volume based on the manifest and
individual chunks referenced in the manifest.

To support incremental backup functionality, we introduce another object called
shafile to the list of backup files. shafile file helps track changes to the
volume since last backup. This new object holds SHA256s of the
volume. A brief description of SHA-2 or SHA256 can be found at
http://en.wikipedia.org/wiki/SHA-2. The backup manifest file will have a
reference to this object. During a full backup operation, Cinder divides up the
volume into fixed blocks of user configurable block size. It calculates SHA256
of each block and compiles a list of SHAs and uploads the shafile to the backup
container.

To keep the incremental backup implementation simple, an incremental operation
is only performed with respect to a full backup. During incremental backup,
cinder reads the shafile of the full backup. It creates a new shafile from
the current volume data and compares the new shafile with full backup shafile
to calculate the blocks that are changed since last full backup.

We will use existing manifest mechanism to capture the delta. Since
full backups do not contain any holes, offset+lengths of each chunk of
the volume describe the full length of the volume logical address. However
with incremental backup, this model is challenged and the offset/chunk of
individual files become sparse. The absence of offset/length in a manifest
represents the data that is not modified since last backup. One potential
drawback of this approach is if changes to volume are fragmented, incremental
backup may result in too many objects in Swift. However object stores
like Swift are built to handle many small objects effectively.

The new shafile is uploaded as part of the incremental backup.

The manifest header identifies this backup as incremental backup and hence
contains a reference to the full backup container.

Following changes are made to the manifest header of the backup

::

        metadata['version'] = self.DRIVER_VERSION
        metadata['backup_id'] = backup['id']
        metadata['volume_id'] = volume_id
        metadata['backup_name'] = backup['display_name']
        metadata['backup_description'] = backup['display_description']
        metadata['created_at'] = str(backup['created_at'])

        # Changes to metadata section of manifest
        metadata['shafile'] = <shafilename>  # Path to shafile name. Or
                                             # can be hardcoded to "shafile"
                                             # in the container

        metadata['backup_type'] = "incrementa/full" # backup type
        metadata['full_container'] = <object path> # path of full backup

Restore API is not expected to change, however restore implementation will be
changed to handle incremental backup. To keep the restore from incremental
backup simple and easy to test, the restore operation first performs restore
of the full volume from the full backup copy and then apply incremental
changes at offset and length as described in the incremental backup manifest.


Snapshot based backups::

 Since existing backup implementation copies the data directly from the volume,
 it requires the volume to be detached from the instance. For most cloud
 workloads this may be sufficient but other workloads that cannot tolerate
 prolonged downtimes, a snapshot based backup solution can be a viable
 alternative. Snapshot based backup will perform a point in time copy of the
 volume and backup the data from point in time copy. This approach does not
 require volume to be detached from the instance. Rest of the backup and
 restore functionality remain the same.

 As an alternative, snapshot based backup can be implemented by extending
 existing backup functionality to snapshot volumes. This approach can be lot
 more simpler than backup API taking snapshot of the volume and then managing
 the snapshots.

Alternatives
------------
Incremental backup offers two important benefits:
 1. Use less storage when storing backup images
 2. Use less network bandwidth and improve overall efficiency of backup process
    in terms of CPU and time utilization

The first benefit can be achieved as a post processing of the backup images to
remove duplication or by using dedupe enabled backup storage. However the
second benefit cannot be achieved unless Cinder backup supports incremental
backup.

Data model impact
-----------------
No percieved data model changes

REST API impact
---------------
No new APIs are proposed. Instead existing backup API will be enhanced to
accept additional option called "--incr" with <path to full backup container>"
as its argument.

::

 cinder backup-create <volumeid> --incr <full backup container>
   Performs incremental backup

 cinder backup-create <volumeid> --snapshot
   Optionally backup-create will backup a snapshot of the volume. Snapshot
   based backups can be performed while the volume is still attached to the
   instance.

 cinder backup-create <volumeid> --snapshot --incr <full backup container>
   Optionally backup-create will perform incremental backup from volume
   snapshot

No anticipated changes to restore api

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
python-cinderclient will be modified to accept "--incr" option. It may
include some validation code to validate if the full backup container path
is valid

Currenly backup functionality is not integrated with OpenStack dashboard. When
it happens, the dashboard will provide an option for user to choose incremental
backup

Performance Impact
------------------
Except for calculating SHAs during full backup operation, there is no other
performance impact on existing API. The performance penalty can be easily
offset by the efficiency gained by incremental backup. Also new hardware
support CPU instructions to calculate SHAs which alleviates some stress on
the CPU cycles.

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
muralibalcha(murali.balcha@triliodata.com)

Other contributors:
giribasava(giri.basava@triliodata.com)

Work Items
----------
1. python-cinderclient
   That accepts "--incr" option and some validation code

2. cinder/api
   Which parses the "--incr" option

3. cinder/backup/api.py
   backup api signature is modified

4. cinder/backup/manager.py

5. cinder/backup/driver/swift.py
   Heavy lifting is done here.
   Both backup and restore apis will be modified.

Dependencies
============

None

Testing
=======

Unit tests will be added for incremental backup.

Testing will primarily focus on the following:
 1. SHA file generation
 2. Creating various changes to the original volume. These include

  1. Changes to first block
  2. Changes to last block
  3. Changes to odd number of successive blocks
  4. Changes to even number of successive blocks
  5. Changes spread across multiple sections of the volume

 3. Perform 1 incremental
 4. Peform multiple incremental backups
 5. Restore series of incremental backups and compare the contents
 6. Perform full backup, then incremental, then full and then incremenal
    restore the volume from various backups.

Documentation Impact
====================

Need to document new option in the block storage manual.

References
==========

None

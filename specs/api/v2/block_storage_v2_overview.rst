=============================
Block Storage API v2 Overview
=============================

OpenStack Block Storage is a block-level storage solution that enables
you to:

-  Mount drives to OpenStack Cloud Servers™ to scale storage without
   paying for more compute resources.

-  Use high performance storage to serve database or I/O-intensive
   applications.

You interact with Block Storage programmatically through the Block
Storage API as described in this guide.

Note
~~~~

-  OpenStack Block Storage is an add-on feature to OpenStack Nova
   Compute in Folsom versions and earlier.

-  Block Storage is multi-tenant rather than dedicated.

-  Block Storage allows you to create snapshots that you can save, list,
   and restore.

-  Block Storage allows you to create backups of your volumes to Object
   Storage for archival and disaster recovery purposes. These backups
   can be subsequently restored to the same volume or new volumes.

Concepts
--------

To use the Block Storage API effectively, you must understand several
key concepts:

-  **Volume**

   A detachable block storage device. You can think of it as a USB hard
   drive. It can only be attached to one instance at a time.

-  **Volume type**

   A type of a block storage volume. You can define whatever types work
   best for you, such as SATA, SCSCI, SSD, etc. These can be customized
   or defined by the OpenStack admin.

   You can also define extra\_specs associated with your volume types.
   For instance, you could have a VolumeType=SATA, with extra\_specs
   (RPM=10000, RAID-Level=5) . Extra\_specs are defined and customized
   by the admin.

-  **Snapshot**

   A point in time copy of the data contained in a volume.

-  **Instance**

   A virtual machine (VM) that runs inside the cloud.

-  **Backup**

   A full copy of a volume stored in an external service. The service
   can be configured. The only supported service for now is Object
   Storage. A backup can subsequently be restored from the external
   service to either the same volume that the backup was originally
   taken from, or to a new volume.


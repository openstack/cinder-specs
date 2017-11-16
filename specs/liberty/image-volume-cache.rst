..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Generic image cache functionality
==========================================

https://blueprints.launchpad.net/cinder/+spec/image-volume-cache

Currently some volume drivers implement the clone_image method and use an
internal cache of volumes on the backend that hold recently used images. For
storage backends that can do very efficient volume clones it is a potentially
large performance improvement over having to attach and copy the image
contents to a each volume. To make this functionality easier for other volume
drivers to use, and prevent any duplication in the code base, we should
implement a more unified way to do this caching.


Problem description
===================

If we don't create a unified approach to this image caching we will end up
with multiple vendors implementing it in volume drivers. This means duplicated
code, redundant vendor prefixed config options, and a non-uniform way for
admins to set up and configure image caching.

Use Cases
=========

The primary (and I think only) use case for this is when creating a volume
from an image more than once. As an end user I will see (potentially) faster
volume creation from an image after the first time, and as an admin once I
set this up once in the backend configuration it should require no more
interaction.

Proposed change
===============

When creating a volume from an image using the cache there are a few high
level steps to take:

* Check if an image has a corresponding entry in the cache.
* If it is in the cache but has been updated, delete it.
* If it wasn't in the cache (or isn't anymore)
    * Evict an old image entry if room is needed.
    * Create a volume from an image (the 'normal' way of copying data over),
      this volume will henceforth be known as the 'image-volume'. This volume
      would be owned by a special tenant that is controlled by Cinder.
    * Update the cache to know about this new image-volume and its image
      contents.
* Clone the image-volume.

This new behavior would be enabled via a new volume driver config option
called 'image_volume_cache_enabled'. The size of this cache will then be
defined by a few new config options:

* image_volume_cache_max_size_gb
* image_volume_cache_max_size_percent
* image_volume_cache_max_count

These options are scoped to each backend.

In the _create_from_image of CreateVolumeFromSpecTask we can add in the logic
to check first if the image cache is enabled for the target backend. If it is
then we can use the cache, if not we use the slow path only. This would be
done after driver.clone_image is called to give a backend a chance to do an
even more optimal clone if possible.

For the actual image-volume I think it would be best if we can re-use the
normal Cinder Volume model and database table. Ideally we can leverage the
cinder-internal tenant feature so that the special tenant will own these
cached images.

We then would need a new table to track our image cache. This table will have
Volume, Image, and Host id's, as well as image meta-data to make a decision
as to whether or not it is still 'valid'. More details in the Data Model
Impact section.

This information gives us enough to look up whether or not a image is in the
cache for a target backend, and make the decision to either create a new one
or not and then call _create_from_source_volume with the image-volume.

As far as the volume driver is concerned it would be creating volumes from an
image and cloning volumes, it would not need to be aware of the cache at all
for the creation methods.

For any backends which need to use snapshots they will have to transition from
image-volume -> snapshot -> volume. In the future this image cache table could
always be expanded to have a snapshot_id in addition to the volume_id, but for
the first iteration of this it will only deal with volumes as the cache
backing.

There is a possibility of a race within the cache where an image-volume would
be getting evicted and is selected to be used for a clone operation. If this
happens we should catch the error and attempt to create the volume by
downloading the image. It is also possible that multiple of the same images
are cached at the same time, this is considered 'OK', eventually extra ones
will be evicted if space is needed.

In future iterations of this we could expose the objects in the cache as
public snapshots, and store the backing volume for other public snapshots
in the same way we do the cached objects. This should allow for quite a bit
of re-use in the code, and give users another way to get quick copies of
volumes for backends with optimized snapshot->volume and volume->volume
operations. In the first iteration this functionality will not be provided.
It is worth noting so that we can ensure the implementation is extensible to
allow for it in the future.


Alternatives
------------

A very simple alternative to this is to simply push all this logic and
behavior into each volume driver and live with the duplicated code and config
options. This however prohibits backends that cannot store meta-data for their
volumes anywhere.

Another alternative is to not use the actual Volume objects and to expand the
image cache table to have enough information for a backend to create volumes
and clones from it (maybe metadata type values?). Then we could add new API's
to the volume driver such as 'create_cached_image_volume',
'delete_cached_image_volume', 'create_volume_from_cached_image', and so on.
This would put more of the logic into the volume drivers, but requires them to
implement these new methods. The benefit here is that we don't start having
special 'internal' volumes. The downside being that each driver then has to
implement all of these methods which would do almost the same thing as
create_volume, delete_volume, etc, but with different parameters.

Data model impact
-----------------

A new database table for the image cache.

::

    +------------+-----------+--------------------------------+
    |    NAME    |    TYPE   |           DESCRIPTION          |
    +------------+-----------+--------------------------------+
    |            |           |                                |
    | id         |  String   |  Auto generated UUID           |
    |            |           |                                |
    | updated_at |  DateTime |  The updated_at time from the  |
    |            |           | image metadata                 |
    |            |           |                                |
    | host_id    |  String   |  ForeignKey for Host.id that   |
    |            |           | has the backing volume         |
    |            |           |                                |
    | volume_id  |  String   |  ForeignKey for Volume.id of   |
    |            |           | the backing volume             |
    |            |           |                                |
    | image_id   |  String   |  The image id from the image   |
    |            |           | metadata                       |
    +------------+-----------+--------------------------------+

REST API impact
---------------

None

Security impact
---------------

The special Cinder owned tenant could potentially be a risk if someone was able
to get a hold of its credentials or access the image-volumes. Worst case
someone could alter the cached image volumes if they had permission to attach
and write to them directly.

Care will have to be taken to ensure it isn't accessible by normal users.

Notifications impact
--------------------

New info log messages and event notifications for whether the cache hit
or missed. That way there is enough info to determine how effective it is and
if settings need to be adjusted.

Other end user impact
---------------------

None

Performance Impact
------------------

This should improve performance on average for systems that can do efficient
volume clones when doing create volume from image. There will be many factors
involved as to how much it changes, but it is unlikely to be much slower.

It is possible this will add some delay on occasional requests which hit a
'worst case' scenario of having to do the database lookups, trying to create
from a cached image, failing because it got evicted, and then doing the image
download. In situations where that occurs frequently the cache size could be
modified or the feature disabled.

Other deployer impact
---------------------

New configuration options for a cinder backend that would potentially need to
be set:

* image_volume_cache_enabled (Boolean)
* image_volume_cache_max_size_gb (Integer)
* image_volume_cache_max_size_percent (Integer)
* image_volume_cache_max_count (Integer)

Developer impact
----------------

Just new DB API's and tables to be aware of.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  patrick-east

Other contributors:
  None

Work Items
----------

* DB changes
* create_volume flow changes


Dependencies
============

None


Testing
=======

* DB migration tests
* Unit tests for DB API's
* Unit tests for flow changes


Documentation Impact
====================

New configuration options.


References
==========

* https://blueprints.launchpad.net/cinder/+spec/cinder-internal-tenant

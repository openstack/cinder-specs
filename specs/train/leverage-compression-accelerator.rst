..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Leverage compression accelerator
================================

https://blueprints.launchpad.net/cinder/+spec/leverage-compression-accelerator

This blueprint proposes to leverage hardware compression accelerator to
accelerate the image compression when uploading volume to glance as image.

Problem description
===================

When trying to upload volume to glance as image, currently all the format
transformation is done by software and the performance is not good.

For example, when uploading a volume of size 30GB with 12GB data resides,
select qcow2 as the target image format, currently the tool used to do the
transformation is qemu-img. This has below problems:

#. Low compression rate
   After transforming, the output size is 12GB.

#. High CPU consuming
   During the transforming, CPU usage is about 100%-170% (~ two cores).

#. Long duration
   The time to transform the volume is about 23 seconds.

As a contrast, if using hardware accelerator to do the format transformation,
choose 'gzip' as the target format, the performance is much better. Take Intel
QAT as the accelerator for example, performance data would be:

#. Compression rate
   After transforming, the output size is 2.3GB.

#. CPU consuming
   During the transforming, CPU usage is about 97% (one core).

#. Duration
   The time to transform the volume is about 9 seconds.

So the compressed size is only a quarter of its original size, but takes less
than half the time.

Use Cases
=========

* User wants to upload volume to glance as a compressed image

Proposed change
===============

- Add a new container format 'compressed'

  Cinder don't have an enum to control container format list, rather than disk
  format. But cinder will have specific steps for the 'compressed' container
  format. Refer to below changes.

  The disk format remains unchanged by this spec, it can be any type in enum in
  cinder.api.schemas.volume_actions.volume_upload_image.

- Add an option to enable/disable the image compression support

  If there's no hardware accelerator exists, then it will fallback to software
  solution to do the compression. But this will consume CPU resource. So this
  spec will add an option to control if the compression feature be enabled or
  not.

- Add compression tool of famous hardware accelerators into file volume.filters

  The compression tool such as 'qzip' will be added in volume.filters so that
  it can be run by root wrapper

- Show 'compressed' in the container format list when selecting a volume to
  upload

  This need to be done in cinder client and horizon.

- Check if there are some hardware accelerators exist in the system

  Go through the accelerator tool list to find if any hardware accelerator
  exist. If none exist, failover to software solution if compressed format
  is selected.

  This will be done in ./cinder/image/accelerator.py, and at last, will call
  the driver of accelerator to check the existence.

  Refer to:
  https://review.opendev.org/#/c/668825/1/cinder/image/accelerator.py
  Function: is_engine_ready
  https://review.opendev.org/#/c/668825/1/cinder/image/accelerators/qat.py
  Function: is_accel_exist

- Do image compression after convert_image but before uploading to glance

  Call the existing accelerator's tool to do the compression. e.g. if Intel QAT
  device is identified, then call qzip here to do the compression. If no
  accelerator identified, then fallback to software solution, using the tool
  gzip to do the decomression.

- Do image decompression after downloading from glance but before convert_image

  When creating volume from image, call hardware accelerator to do the
  decompression. If hw accelerator doesn't exist, then fallback to software
  solution, using the tool gzip to do the decomression

Alternatives
------------

Another way is to use the improved processor with software solution only, such
as gzip. But the cost would be high vs dedicated hardware accelerator, and have
no performance benefit.

REST API impact
---------------

Add the image type of "compressed" when uploading volume to image

Data model impact
-----------------

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

* Less CPU comsuming if hardware accelerator available
* Less conversion time if hardware accelerator available
* Smaller image size

Other deployer impact
---------------------

* Need to install the driver and utility tool of the accelerator
* Can configurate the option to enable/disable compression capability

Developer impact
----------------

* Developers that want to add support for other accelerators need to add
  their tool to root wrap

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Liang Fang <liang.a.fang@intel.com>

Work Items
----------

* During image conversion, switch to hardware accelerator if possible
* Implement a common framework for hw leverage in image conversion
* Implement a typical hw accelerator in image conversion
* Unit test be added

Dependencies
============

None

Testing
=======

* Unit-tests, tempest and other related tests will be implemented.
* Test case in particular: when creating a volume from an image with
  container_format == 'compressed', but the image contains some format other than
  what Cinder supports. The expected behavior is to fail gracefully.

Documentation Impact
====================

* Documentation will be needed. User documentation on how to use accelerator
  and developer documentation on how to add an additional accelerator

References
==========

_`[1]` https://review.opendev.org/#/c/668825/
_`[2]` https://review.opendev.org/#/c/668943/

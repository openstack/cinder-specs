..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Support modern compression algorithms in cinder backup
======================================================

https://blueprints.launchpad.net/cinder/+spec/support-modern-compression-algorithms-in-cinder-backup

This blueprint proposes to add support of modern compression algorithms like
zstd to cinder backup.

Problem description
===================

Cinder backup currently supports only gzip and bzip2 to compress/decompress the
volume data during backup and restore operation.

It should support other compression algorithms as well because efficiency of
compression entirely depends on cloud provider's environment and choice. Some
may want fast compression compromising the compression ratio, so to provide more
choices to cloud provider, it will be good to add support of other algorithm
like zstd.

Zstandard(zstd) gives high compression ratio(comparable to zlib) with very much
fast speed. It's now used as package compression algorithm for arch linux.

https://facebook.github.io/zstd/

Some general benchmark result is available on the website introduced above, or
lzbench website.

https://github.com/inikep/lzbench/

Use Cases
=========

* Cinder backup with high-speed compression algorithm

Proposed change
===============

* add zstd algorithm package details in requirements.txt.

* add the support of zstd algorithm in get compressor routine of cinder backup.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

* Cinder backup and restore operation gets faster.
* Size of backup data may be bigger than zlib or bz2 compression algorithm
  (depends on content of data volume)


* Simple performance test result I have done

  * Server spec: 8core CPU, 16GB memory
  * Backup store is NFS-backend on localhost (i.e. small network bottleneck)
  * Backup/restore a 16GB volume created from a Windows image (qcow2
    12001017856 bytes)
  * Measured backup/restore time backup data size, and size ratio versus size of
    qcow2 (considering as approximate amount of data in the volume)
    The smaller the better for all values.
  * CPU usage for algo other than zstd is about 100% to 110% (i.e. Using only
    one core.) where max usage is 800% (= 8cores * 100%).
  * CPU usage for zstd is about 80% to 400% (utilizes multi cores).
  * Famous algorithm like gzip may performs better if offloaded to hardware.

================== ================ ================= =========== =============
Algorithm          backup time(sec) restore time(sec) size(bytes) v.s. qcow2(%)
================== ================ ================= =========== =============
none                          205.8             109.2 17218167818         143.5
gzip                          743.2             141.5  6658990044          55.5
bz2                          2102.5             742.2  6480248772          54.0
zstd(to be added)             196.2             100.2  6286227236          52.4
lz4(for reference)            211.0             113.0  7903490364          65.9
================== ================ ================= =========== =============

Other deployer impact
---------------------

The python module `zstd <https://pypi.org/project/zstd/>`_ is required to use
zstd compression algorithm.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee: Kazufumi Noto

Other contributors: None

Work Items
----------

* Add the package details to cinder/requirements.txt.
* Implement the code to add support of zstd compression.
* Documents need to be updated to reflect the added support.
* Add test cases to create backup using zstd compression.

Dependencies
============

None

Testing
=======

* Unit tests will be added to create backup using zstd compression.

Documentation Impact
====================

The following documents need to be updated to reflect the added support:

* cinder sample configuration file in configuration reference.

References
==========

* https://facebook.github.io/zstd/

* https://github.com/inikep/lzbench/

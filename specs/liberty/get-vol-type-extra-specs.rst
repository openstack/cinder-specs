..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Retrieve Supported Volume Type Extra Specs
==========================================

https://blueprints.launchpad.net/cinder/+spec/get-volume-type-extra-specs

Provide an interface to obtain a volume driver's *capabilities*.

Definitions
===========

* *Volume Type:* A group of volume policies.
* *Extra Specs:* The definition of a volume type. This is a group of policies
  e.g. provision type, QOS that will be used to define a volume at creation
  time.
* *Capabilities:* What the current deployed backend in Cinder is able to do.
  These correspond to extra specs.

Problem description
===================

The current implementation of *volume type* *extra specs* management process in
Horizon and the cinder client are error-prone. Operators manage extra specs
without having guidance on what capabilities are possible with a backend.
Without knowing the capabilities, the operator doesn't know what the possible
*extra spec* key/values are.

Today operators have to make sure they're reading the right documentation that
corresponds to the version of their storage backend. Documentation can
become out of date with their volume driver.

It would be more convenient, and a better user experience if the operator was
able to ask Cinder for the capabilities of the current deployed storage
backends, instead of having to guess.

Use Cases
=========

As an operator, I want to get a list of the capabilities for my deployed
storage backends in Cinder.

With these capabilities, I have keys and possible values that can be used
within a volume type's extra specs.

Potentially with the API endpoint, Horizon could use it to improve the GUI for
managing extra specs.

Proposed change
===============

Cinder volume drivers will implement a new method ``get_capabilities``, which
will return a dictionary of supported capabilities from the storage backend.

The dictionary will include some brief information about the backend, and
capabilities that correspond to extra spec keys and values.

Features like create volume, create snapshots, etc are considered minimum
features [1]. Unlike ``well defined`` keys [2], minimum features are required
to implement. With ``well defined`` keys, drivers just need to report if the
capability is not supported.
If the backend has some vendor unique capabilities, the backend driver can
also define ``vendor unique`` keys for supported capabilities.

Alternatives
------------

Operators could continue to guess what extra specs they should be setting in
volume types. Operators could attempt to find documentation for volume type
extra specs. As a last resort, operators could dig through different Cinder
volume driver code to figure out the possible extra types. These aren't
reliable solutions.

Data model impact
-----------------

None

REST API impact
---------------

New endpoint GET /v2/tenant/get_capabilities/ubuntu@lvm1_pool::

 {
   "namespace": "OS::Storage::Capabilities::ubuntu@lvm1_pool",
   "volume_backend_name": "lvm",
   "pool_name": "pool",
   "driver_version": "2.0.0",
   "storage_protocol": "iSCSI",
   "display_name": "Capabilities of Cinder LVM driver",
   "description": "These are volume type options provided by Cinder LVM driver, blah, blah.",
   "visibility": "public",
   "properties": {
    "thin_provisioning": {
       "title": "Thin Provisioning",
       "description": "Sets thin provisioning.",
       "type": "boolean"
     },
     "compression": {
       "title": "Compression",
       "description": "Enables compression.",
       "type": "boolean"
     },
     "vendor:compression_type": {
       "title": "Compression type",
       "description": "Specifies compression type.",
       "type": "string",
       "enum": [
         "lossy", "lossless", "special"
       ]
     },
     "replication": {
       "title": "Replication",
       "description": "Enables replication.",
       "type": "boolean"
     },
     "qos": {
       "title": "QoS",
       "description": "Enables QoS.",
       "type": "boolean"
     },
     "vendor:minIOPS": {
       "title": "Minimum IOPS QoS",
       "description": "Sets minimum IOPS if QoS is enabled.",
       "type": "integer"
     },
     "vendor:maxIOPS": {
       "title": "Maximum IOPS QoS",
       "description": "Sets maximum IOPS if QoS is enabled.",
       "type": "integer"
     },
     "vendor:minIOPS": {
       "title": "Burst IOPS QoS",
       "description": "Sets burst IOPS if QoS is enabled.",
       "type": "integer"
     },
     "vendor:persona": {
       "title": "Persona",
       "description": "I am something..." ,
       "default": "Generic",
       "enum": [
         "Generic",
         "ONTAP-legacy",
         "VMware",
         "OpenVMS",
         "HPUX",
         "WindowsServer",
         "Generic-ALUA",
         "Generic-legacy",
         "HPUX-legacy",
         "AIX-legacy",
         "EGENERA"]
     }
   }
 }

The ``well defined`` keys are indicated without a prefix like the "qos".
These are fairly standard base keys for Cinder backends. We expect most
devices to report at least a boolean True/False for these keys.

The ``vendor unique`` keys are optional and are indicated with a prefix
of vendor name + colon(:). (ex. abcd:minIOPS)
Vendor driver can use anything for the ``vendor unique`` keys, but the
vendor name prefix shouldn't contain colon because of the separator and
it will be automatically replaced by underscore(_). (abc:d -> abc_d)

Let's look at compression here:
  This is a ``well defined`` key, we expect devices to report True or False
  regarding whether they support it or not. In the case where not only does
  a device support it, but it can be configured, the option keys are listed
  under the "options" portion. This is simply the <key-name> of the option,
  and a list of valid values that can be specified for it. NOTE, if the options
  key is empty ({}) that means there are NO options that can be set on that
  capability key.

The vendor:fireproof capability:
  This is a ``vendor unique`` key, and is indicated by being prefixed with
  "vendor name" + ":". Also, note that the default is True and that there
  are NO options. The example indicates that this device is ALWAYS fireproof,
  you can't change that, it just is, what it is.

The thin_provisioning capability:
  This is a ``well defined`` key which is not supported by this particular
  vendor. As a result, it defaults to False, and provides no options.

The qos capability describes some corner cases for us:
  This key is a ``well defined`` key, that's very customizable via options.
  Well defined portion is whether the capability is supported or not (again
  True/False), again, some devices may allow deploying volumes with or without
  QoS on the same device, so you can specify that with
  <capability-key-name>=true|false.

  If a device doesn't support this (ie always true), then this entry is omitted.

  The other key piece is ``vendor unique`` keys. For those that allow
  additional special keys to set QoS those key names are provided in list
  format as valid keys that can be specified and set as related to Quality of
  Service.

The vendor:persona key is another good example of a ``vendor unique`` capability:
  This is very much like QoS, and again, note that we're just providing what
  the valid values are.

  You'll notice that the data-structure follows the settings you would put in
  your extra-specs. This particular case doesn't have any options other than
  the base key itself, so the structure is rather simple.

Security impact
---------------

The endpoint mentioned in the API Impact will only be available through the
``admin_api`` policy. Operators or other OpenStack services will have the
ability to query this information.

Notifications impact
--------------------

None

Other end user impact
---------------------

Cinder Client Example:

The operator wants to define new volume types for their OpenStack cloud. The
operator would fetch a list of capabilities for a particular backend's pool:

First get list of services::

  $ cinder service-list
  +------------------+-----------------+------+---------+-------+----------------------------+-----------------+
  |      Binary      |    Host         | Zone |  Status | State |         Updated_at         | Disabled Reason |
  +------------------+-----------------+------+---------+-------+----------------------------+-----------------+
  | cinder-scheduler | controller      | nova | enabled |   up  | 2014-10-18T01:30:54.000000 |       None      |
  | cinder-volume    | block1@lvm#pool | nova | enabled |   up  | 2014-10-18T01:30:57.000000 |       None      |
  +------------------+-----------------+------+---------+-------+----------------------------+-----------------+

With one of the listed pools, pass that to capabilities-list::

  $ cinder capabilities-list block1@lvm#pool

  host_name: block1
  volume_backend_name: lvm
  pool_name: pool
  driver_version: 2.0.0
  storage_protocol: iSCSI

  capabilities:

    compression:
      default: true
      options:
        compression: true, false
        compression_type: lossy, lossless, special

    thin_provisioning:
      default: false

    replication:
      default: false

    qos:
      default: true,
      options:
        qos: true, false
        vendor_keys:
          vendor:minIOPS,
          vendor:maxIOPS,
          vendor:burstIOPS

    vendor:fireproof:
      default: true
      options: {}

    vendor:persona:
      default: Generic
      options:
        Generic
        ONTAP-legacy
        VMware
        OpenVMS
        HPUX
        WindowsServer
        Generic-ALUA
        Generic-legacy
        HPUX-legacy
        AIX-legacy
        EGENERA


Horizon:

Horizon will be updated to include the displaying of the supported capabilities
so operators can select and set the values while creating or editing the
volume type extra specs.

If the volume type does not have any volume backend name associated with it,
Horizon will not have any extra specs keys to display. Administrators can still
enter in key/value pairs of their own. This is the same behavior as the current
process.

If a driver does not publish the ``extra specs`` dictionary, which will be the
case for any drivers that do not get updated, then no client-side filtering
will be performed, and the behavior will basically revert to the current
situation where the administrator in horizon will need to know and enter the
key/value pairs without any additional guidance.


Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Driver maintainers would need to implement a method ``get_capabilities``. This
should fetch from the storage backend a list of capabilities and translate it
to the dictionary structure::

 {
   'driver_version:' '2.0.0',
   'storage_protocol:' iSCSI,
   'capabilities:' {
     'compression': {
       'default': True,
       'options': {
         'compression_type': ['lossy', 'lossless', 'special'],
         'compression': [True, False]
       }
     },
     'thin_provisioning': {
       'default': True,
       options: {
         'thin_provisioning': [True, False]
       }
     },
     'qos': {
       'default': True,
       options: {
         'qos': [True, False],
     }
    }
    'replication': {
      'default': True,
      options: {
        replication: [True, False]
      }
    }
  }

There's nothing keeping a vendor reporting fewer or more keys, but the
following are strictly enforced:

* The data structure
* The information in the capabilities
* The ``well defined`` capabilities.
* Driver version
* Storage protocol

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * jgravel (julie.gravel@hp.com)

Other contributors:
  * gary-smith (gary.w.smith@hp.com)
  * thingee (thingee@gmail.com)
  * mtanino (mitsuhiro.tanino@hds.com)

Work Items
----------

* Add new endpoint to Cinder API.
* Add RPC call.
* Add new volume manager method for get_capabilities.
* Update LVM reference implementation with get_capabilities method.

Dependencies
============

The decision on what the ``well defined`` capabilities will be [2].

Testing
=======

* Unit tests
* Eventually, tempest tests once all drivers are supporting it.

Documentation Impact
====================

Update the Cinder developer documentation for driver maintainers to reference
how to push capabilities from their volume driver.

References
==========

* [1] - http://docs.openstack.org/developer/cinder/devref/drivers.html#minimum-features
* [2] - https://review.openstack.org/#/c/150511
* Related horizon spec: https://blueprints.launchpad.net/horizon/+spec/vol-type-extra-specs-describe

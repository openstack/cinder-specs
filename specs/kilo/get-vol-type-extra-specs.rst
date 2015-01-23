..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Retrieve Supported Volume Type Extra Specs
==========================================

https://blueprints.launchpad.net/cinder/+spec/get-volume-type-extra-specs

The purpose of this spec is to modify the driver's ``get_volume_stats`` method
to include in its return value a dictionary of volume type extra spec keys that
each driver supports.  The dictionary will be used to improve the volume types
extra spec management process in Horizon.

Problem description
===================

The current implementation of volume type extra specs management process in
Horizon is error-prone. Administrators must manage extra specs without having
any interactive guidance on what possible keys are available and what their
values are. Having the ability to obtain the permissible volume type extra
specs while creating or editing the extra specs will make this task easier and
more user-friendly.

Use Cases
=========

Proposed change
===============

The driver's ``get_volume_stats`` method will be modified to also return a
dictionary of supported volume type extra spec keys. The dictionary will
include information about those keys such as key description, possible values,
value range, and value type. Once the get_volume_stats() method was modified,
Horizon could call the existing ``scheduler_stats`` API call to get the
dictionary and use it in the extra specs creating or editing process. The new
``extra_specs`` dictionary will be included within the existing
``capabilities`` dictionary:

Current json response of the ``scheduler_stats`` API call::

  {
    "pools": [
      {
        "name": "ubuntu@lvm#backend_name",
        "capabilities": {
          "pool_name": "backend_name",
          "QoS_support": false,
          "location_info": "LVMVolumeDriver:ubuntu:stack-volumes:default:0",
          "timestamp": "2014-11-21T18:15:28.141161",
          "allocated_capacity_gb": 0,
          "volume_backend_name": "backend_name",
          "free_capacity_gb": 7.01,
          "driver_version": "2.0.0",
          "total_capacity_gb": 10.01,
          "reserved_percentage": 0,
          "vendor_name": "Open Source",
          "storage_protocol": "iSCSI"
        }
      }
    ]
  }


New json response of the ``scheduler_stats`` API call::

  {
    "pools": [
      {
        "name": "ubuntu@lvm#backend_name",
        "capabilities": {
          "QoS_support": false,
          "allocated_capacity_gb": 0,
          "driver_version": "2.0.0",
          "free_capacity_gb": 7.01,
          "location_info": "LVMVolumeDriver:ubuntu:stack-volumes:default:0",
          "pool_name": "backend_name",
          "reserved_percentage": 0,
          "storage_protocol": "iSCSI",
          "timestamp": "2014-11-21T18:15:28.141161",
          "total_capacity_gb": 10.01,
          "vendor_name": "Open Source",
          "volume_backend_name": "backend_name",
          "extra_specs": [
            {
              "description": "HP 3PAR Fibre Channel driver",
              "display_name": "HP3PARFCDriver",
              "namespace": "OS::Cinder::HP3PARFCDriver",
              "properties": {
                "hp3par:persona": {
                  "default": "2",
                  "description": "HP 3PAR Persona",
                  "enum": {
                    "1": "Generic",
                    "10": "ONTAP-legacy",
                    "11": "VMware",
                    "12": "OpenVMS",
                    "13": "HPUX",
                    "15": "WindowsServer",
                    "2": "Generic-ALUA",
                    "6": "Generic-legacy",
                    "7": "HPUX-legacy",
                    "8": "AIX-legacy",
                    "9": "EGENERA"
                  },
                  "title": "persona",
                  "type": "string"
                },
                "hp3par:snap_cpg": {
                  "description": "Overrides the 'hp3par_snap' setting",
                  "title": "snap_cpg",
                  "type": "string"
                }
              }
            },
            {
              "description": "HP LeftHand iSCSI driver",
              "display_name": "HPLeftHandISCSIDriver",
              "namespace": "OS::Cinder::HPLeftHandISCSIDriver",
              "properties": {
                "hplh:provisioning": {
                  "default": "thin",
                  "description": "Provisioning",
                  "enum": [
                    "full",
                    "thin"
                  ],
                  "title": "provisioning",
                  "type": "string"
                },
                "hplh:vvs": {
                  "default": "1",
                  "description": "VVS",
                  "title": "vvs",
                  "type": "integer"
                }
              }
            }
          ]
        }
      }
    ]
  }


The properties attributes are defined using simple JSON schema notation. Please
refer to the `JSON Schema documentation`_ for a complete definition.


Alternatives
------------

The alternative of this proposal is the current Horizon process of managing the
volume type extra specs which does not provide any helpful information to
administrators regarding possible extra specs keys and values. They have to
know the exact spellings of the key/value pair that they want to set for each
volume type ahead of time. Most of the time they have to look through the
driver documentation or even the code to see what keys could be used in their
situation.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

Horizon will be updated to include the displaying of the supported extra spec
keys so users can select and set the values while creating or editing the
volume type extra specs. Horizon will use the information about each key to set
constraints for the value input field, which will help to screen out invalid
values at a certain level.

If the volume type does not have any volume backend name associated with it,
Horizon will not have any extra specs keys to display. Administrators can still
enter in key/value pairs of their own. This is the same behavior as the current
process.

If a driver does not publish the ``extra_specs`` dictionary, which will be the
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

Driver developers will need to add a dictionary of volume type extra spec keys
that their drivers support to the return value of the get_volume_stats()
method. The dictionary will contain information about the keys as mentioned in
the proposed change section.

It is completely up to the driver to decide how much information about its
extra spec keys to provide in the dictionary. The driver can also choose not to
provide any extra spec key at all, which means that there would be no extra
specs for that particular driver to display in Horizon. But administrators
would still be able to enter in key/value pairs of their own. This is the same
behavior as the current process.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jgravel (julie.gravel@hp.com)

Other contributors:
  gary-smith (gary.w.smith@hp.com)

Work Items
----------

* Add extra specs dictionary to all supported drivers' ``get_volume_stats``
  method

Dependencies
============

Horizon blueprint that will depend on this spec:

* https://blueprints.launchpad.net/horizon/+spec/vol-type-extra-specs-describe

Testing
=======

Unit tests for all changed files

Documentation Impact
====================

None

References
==========

`JSON Schema documentation`_

.. _JSON Schema documentation: http://json-schema.org/documentation.html

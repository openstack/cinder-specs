..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Failover to alternative iSCSI portals on login failure
======================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/iscsi-alternative-portal

This spec proposes to add fail-over feature to alternative iSCSI portals on
attaching iSCSI volumes when the main iSCSI portal is unaccessible.

Problem description
===================

When the main iSCSI portal is unreachable by network failure etc., volume
attach/detach will fail even though the other portal addresses is reachable.
To enable nova-compute to fall-back to alternative portal addresses, cinder
should tell the alternative portal addresses to nova.

Proposed change
===============

Add optional parameters 'target_alternative_portals',
'target_alternative_iqns', and 'target_alternative_luns' into 'connection_info'
returned by os-initialize_connection volume action API that represents a list
of alternative addresses of the iSCSI portals and the corresponding iqn and lun
to each portal. (iqns and luns can be the same when the addresses are pointing
the same portal.)

When nova-compute recognizes these parameters, it will fall-back to alternative
portal address/target/lun specified by the parameters on failure during
discovery. Old nova-compute just ignores the new parameters, so it works
without changes if the primary portal address is alive.

Note that this proposal is for single iSCSI data path use-case (fail-over).
This means, if the primary portal and target is failed, the other LUNs already
attached on the path will be unaccessible until the path is recovered manually.
Multiple data path (multipath) use-case is covered with another spec.

This change must be applied also to cinder/brick/initiator.

Alternatives
------------

None

Data model impact
-----------------

Backend drivers, if they need, may put multiple portal addresses and iqns into
'provider_location' field of volume tables.

For example, the LVM iSCSI driver will put the addresses in the following
format in order to pass them as a volume dict to target admin helpers:

  provider_location = '<portal ip>:<port>;<portal ip 2>:<port>,<portal> <iqn>'

e.g.

  provider_location = '10.0.1.2:3260;10.0.2.2:3260,0 iqn.2010-10.org.openstack:
                       volume-12345678-abcd-1234-5678-12345678abcd'

(Note that 'volume-12345678...' is a name of target, not a volume; they are
the same in LVM iSCSI driver.)

Other backend drivers can add multiple portal addresses and iqns into
cinder.conf, or can retrieve them dynamically from the array.

REST API impact
---------------

'os-initialize_connection' volume action API response may have additional
parameters 'target_alternative_portals' and 'target_alternative_iqns', which
contain a list of portal IP address:port pairs, a list of alternative iqn's and
lun's corresponding to each portal address. For example:

  {"connection_info": {"driver_volume_type": "iscsi", ...
                       "data": {"target_portal": "10.0.1.2:3260",
                                "target_alternative_portals": [
                                                 "10.0.2.2:3260",
                                                 "10.0.3.2:3260"],
                                "target_iqn": "iqn.2014-2.org.example:vol1-1",
                                "target_alternative_iqns": [
                                              "iqn.2014-2.org.example:vol1-2",
                                              "iqn.2014-2.org.example:vol1-3"],
                                "target_lun": 1,
                                "target_alternative_luns": [2, 3],
                                ...}}}

In this case,
"iqn.2014-2.org.example:vol1-1", lun=1 is for portal "10.0.1.2:3260",
"iqn.2014-2.org.example:vol1-2", lun=2 is for portal "10.0.2.2:3260", and
"iqn.2014-2.org.example:vol1-3", lun=3 is for portal "10.0.3.2:3260".

The portals may have the same targets and/or luns in some backends.

Backend drivers are responsible to specify the order of alternative portals.
Nova and Cinder brick library should first try connecting to the main portal
and target. If failed, it should try connecting to the alternative portals and
targets in the order of the list provided. For example, if a backend driver
wants to round-robin portals for each LUN, the driver may return different
portal addresses as main addresses for LUNs in the same target.


Security impact
---------------

None

Note that if any CHAP credentials are provided in connection_info, they must
be applied on the all target_portals.

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

Backend driver may have additional settings to enable alternative iSCSI
portals. For example, to utilize this feature in iSCSI LVM driver, we needs to
specify a list of alternative IP addresses of the cinder-volume node where
iSCSI targets run on.

Developer impact
----------------

To enable multiple iSCSI portals functionality, backend drivers must change
the implementation of initialize_connection method to return the additional
parameters 'target_alternative_portals', 'target_alternative_iqns' and
'target_alternative_luns'.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tsekiyama

Work Items
----------

- Implement this feature in LVM iSCSI driver as a sample
- Modify Nova and Cinder brick library to fail-over to alternative portals

Dependencies
============

None

Testing
=======

- Unit tests should be added for drivers which support this feature, so that
  initialize_connection will return correct connection_info.

- To test this feature in tempest, multiple addresses must be asigned to the
  test environment in order to access alternative portal addresses.
  Implementation in LVM iSCSI driver would be useful for testing.

Documentation Impact
====================

A section to describe this feature should be added.

If the driver needs additional settings for this feature, the documentation
for them should be added.

References
==========

* nova-specs: Failover to alternative iSCSI portals on login failure
  https://review.openstack.org/#/c/137468/

* cinder-specs: Enhance iSCSI multipath support (multipath use-case)
  https://review.openstack.org/#/c/136500/

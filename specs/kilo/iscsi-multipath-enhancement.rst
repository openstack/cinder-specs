..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enhance iSCSI multipath support
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/cinder/+spec/enhance-iscsi-multipath

This spec proposes to enhance iSCSI multipath support by defining the way to
tell multiple iSCSI portals/iqns/luns to access a volume (LU) to Nova.

Problem description
===================

Currently, nova-compute supports multipath for iSCSI volume data path.
It depends on response to targets discovery of from the main iSCSI portal,
expecting multiple portal addresses are contained.

However, some arrays only respond to discovery with a single portal address,
even if secondary portals are available. In this case, nova-compute cannot know
secondary portals and corresponding iSCSI target IQN, so nova-compute cannot
establish multiple sessions for the target(s). To enable nova-compute to
login to secondary portals, cinder should tell the secondary portal
addresses and corresponding target iqns/luns.

Telling secondary portal addresses and iqns/luns is also useful for arrays
which can respond to discovery with multiple portals addresses and IQNs, since
compute can access to the volume via secondary portals even when the main
portal is unaccessible due to network trouble.
(Note that compute should attach the volume via multipath device "dm-X" so
that the session to the main portal can be re-established when the network
is recovered.)

Use Cases
=========

Proposed change
===============

If nova-compute can support multipath for iSCSI (that is, if multipathd is
installed and CONF.libvirt.iscsi_use_multipath is set True), nova-compute
should call Cinder initialize_connection API with a new 'multipath=True'
optional key in 'connector' dictionary. This parameter is passed to the
backend driver so that the driver can export a volume to multiple ports.

When the 'multipath=True' is specified, 'connection_info' of the API response
will contain multiple portal addresses and corresponding target iqns/luns.
Current 'target_portal'/'target_iqn'/'target_lun' parameters will be replaced
with 'target_portals'/'target_iqns'/'target_luns' which have a list of the
addresses of the iSCSI portals and the corresponding iqn/lun to each portal.
When nova-compute recognizes these parameters, it will log into every
portal address/target/lun. When the backend doesn't support (or not configured
to support) multipath, the driver may return current 'target_portal'/
'target_iqn'/'target_lun' set.

When the 'multipath=False' is specified or the key is omitted, the API must
return 'target_portal'/'target_iqn'/'target_lun' set.

This change must be applied also to cinder/brick/initiator.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

'os-initialize_connection' volume action API will take new optional parameter
'multipath' in the connector argument, that means the connector can handle
multipath to iSCSI volumes.

When the 'multipath=True' is specified, 'os-initialize_connection' volume
action API respond with 'target_portals'/'target_iqns'/'target_luns' in
'connection_info' which contain a list of portal IP address:port pairs,
a list of secondary iqn's and lun's corresponding to each portal address.

For example:

  {"connection_info": {"driver_volume_type": "iscsi", ...
                       "data": {"target_portals": ["10.0.1.2:3260",
                                                   "10.0.2.2:3260"],
                                "target_iqns": [
                                              "iqn.2014-2.org.example:vol1-1",
                                              "iqn.2014-2.org.example:vol1-2"],
                                "target_luns": [1, 2],
                                ...}}}

In this case,
"iqn.2014-2.org.example:vol1-1", lun=1 is for portal "10.0.1.2:3260", and
"iqn.2014-2.org.example:vol1-2", lun=2 is for portal "10.0.2.2:3260".

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

Since nova-compute can omit discovery query to find multiple portals and
targets, this change may make volume-attach/detach operations faster.

Other deployer impact
---------------------

Backend driver may have additional settings to enable iSCSI portal multipath.
For example, to utilize this feature in iSCSI LVM driver, we needs to
specify a list of secondary IP addresses of the cinder-volume node where iSCSI
targets run on.

Developer impact
----------------

If drivers want to use iSCSI multipath, and the connector has the
'multipath=True' value, then drivers should change to return 'target_portals',
'target_iqns' and 'target_luns'.

Existing drivers that support target_portal discovery via the existing
process won't need to change unless they want to honor the 'multipath=True'
connector entry when exporting a volume.

Existing drivers which support multipath with the existing design must work
even if they don't support this change. Such drivers will return single
'target_portal'/'target_iqn'/'target_lun' even if 'multipath=True' is
specified. Then the connector must send discovery query to the returned portal
in order to find the multipath portals and targets, like the existing design.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tsekiyama

Work Items
----------

- Implement this feature in LVM iSCSI driver as a sample
- Enable nova and brick library to handle multiple portals/iqns/luns.

Dependencies
============

None

Testing
=======

- Unit tests should be added for drivers which support this feature, so that
  initialize_connection will return correct connection_info.

- To test this feature in tempest, multiple addresses must be asigned to the
  test environment in order to establish multiple sessions to volumes.
  Implementation in LVM iSCSI driver would be useful for testing.

Documentation Impact
====================

A section to describe this feature should be added.

If the driver needs additional settings for this feature, the documentation
for them should be added.

References
==========

* Enable multipath for libvirt iSCSI Volume Driver (merged)
  https://review.openstack.org/#/c/17946/

* Failover to alternative iSCSI portals on login failure (for single path)
  https://review.openstack.org/#/c/131502/

* Nova-specs: Support iSCSI portal multipath
  https://review.openstack.org/#/c/134299/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Cinder FC Zone Manager - User Friendly Zone Names
=====================================================

https://blueprints.launchpad.net/cinder/+spec/brocade-zone-driver-friendly-zone-names

Introduce user friendly names for zones to include host name and storage
system along with ids to easily identify the host and storage port
details.


Problem description
===================

FC Zone Manager will be enhanced to configure friendly names for
zones. At present, zone manager uses the WWNs of Adapter port and
Target port to configure the zone names.


Use Cases
=========

This addresses the need for zone names to be user friendly so that humans
can easily read the name and understand what host and/or storage system
is represented by that zone.  This feature only has impact on volume
driver developers that need to add the host name and storage system name
into the connection info being passed to the FC zone manager if they wish
to use this feature.


Proposed change
===============

As per current design, the zone names include a default prefix
'openstack' which can be customized and host port wwn and target
wwn (optional) based on the zoning_policy config option:
initiator-target or initiator as follows:

Initiator-Target Zones:
  <openstack><host WWPN><target WWPN>
Initiator Zones:
  <openstack><host WWPN>

These names are not user friendly.

This proposal is to create friendly zone names with host and storage names
as follows:

**Initiator-target zones**
  <Host Name>_<host WWPN>_<storage System>_<target WWPN>

**Initiator zones**
  <Host Name>_<host WWPN>

The required host name and storage system are populated in **connection_info**
dictionary returned by fibre channel volume driver **initialize_connection**
method call. The volume driver will extract the host name from the connector
object passed as parameter to the initialize_connection method. This blueprint
is not documenting the ways the volume driver fetch the storage system value.
It is up to the volume driver to implement a mechanism to retrieve the
storage system value.

Connection Info samples:

{
    'driver_volume_type': 'fibre_channel'
      'data': {
         **'storage_system': 'AMCE_Array',**
         **'host_name': 'OS_Host100',**
         'target_discovered': True,
         'target_lun': 1,
         'target_wwn': '1234567890123',
         'access_mode': 'rw'
      }
}
or
{
    'driver_volume_type': 'fibre_channel'
      'data': {
         **'storage_system': 'AMCE_Array',**
         **'host_name': 'OS_Host100',**
         'target_discovered': True,
         'target_lun': 1,
         'target_wwn': ['1234567890123', '0987654321321'],
         'access_mode': 'rw'
      }
}

On attach/detach, Zone Manager will be initialized with **connection_info**
which is provided by volume driver. Zone Manager in turn will extract the host
name and storage system and will pass this information to the zone driver to
create/delete friendly zone names.

There is a 64 characters limit for zone names. Hence zone driver implements
a mechanism to normalize the host name and storage system to best fit the zone
name. A WWN is 16 characters long and 32 characters are reserved for host and
storage WWNs. For Initiator and Target policy, the max number of characters
available for host name and storage system is 14 characters each. Zone driver
will trim the names to a max of 14 characters. For Initiator only policy,
the max size allowed for host name is 47 characters. Remaining characters
are reserved for '_'.

Zone Driver updates:

Zone driver's add_connection and delete_connection methods will be enhanced
to take host and storage params. These names are defaulted to None. If names
are None, then zone driver will fall back to existing naming mechanism.

The Zone manager will extract the host name and storage system from
connection_info dictionary using host_name and storage_system keys. If keys
are not found or None, then Zone manager will initialize host name and storage
system local variables to None.

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

There is no noticeable performance impact provided FC volume driver will be
able to fetch storage system value swiftly.

Other deployer impact
---------------------

None.

Developer impact
----------------

FC Volume drivers need to enhance their existing drivers to support
friendly zone names. This is optional.

The storage and host name in the **connection_info** object returned by the
**initialize_connection** method of FC Volume driver. See the proposed
change section above for details.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Prakash Kaligotla

Other contributors:
  Nagendra Rao Jaladanki
  Angela Smith

Work Items
----------

- Enhance the zone manager to pass connection_info object to zone driver.
- Implement Brocade Zone Driver to create zones as per new format
using the host name and storage system in the connector object.
- Implement Cisco Zone Driver to create zones as per new format using
the host name and storage system in the connector object.
- Volume drivers are expected to add storage system information to
connector object.
- Unit test the zone driver and client code.


Dependencies
============

Has dependency on FC Volume Drivers to provide host name and storage system
as part of connection_info return dictionary on attach/detach calls.

If driver does not provide host name and storage system, the existing zone
naming mechanism will be used.

Testing
=======

Attach/Detach unit tests will be performed to verify zone manager.


Documentation Impact
====================

None.

References
==========

http://www.brocade.com/downloads/documents/html_product_manuals/FOS_740_CLI/wwhelp/wwhimpl/js/html/wwhelp.htm#href=Title.Fabric_OS.html
http://www.cisco.com/en/US/docs/storage/san_switches/mds9000/sw/rel_2_x/san-os/command/reference/CR02_z.html


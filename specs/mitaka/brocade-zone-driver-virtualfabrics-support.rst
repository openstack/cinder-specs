..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Brocade Cinder Zone Driver - Virtual Fabric Support
===================================================

https://blueprints.launchpad.net/cinder/+spec/brocade-zone-driver-virtualfabrics-support

Introduce Virtual Fabric (VF) support for Brocade FC SAN Switches.


Problem description
===================

As of Juno, there is no support for zone management in Brocade Virtual Fabrics
thus preventing the administrator to add this automated zone access control
in Brocade Virtual Fabrics (VF) environment.


Use Cases
=========

This addresses the case where an end user had the Brocade Virtual Fabrics
feature enabled on the FC fabric and zoning needs to be performed in a VF
other than the default.  The impact of this feature is on end users who now
may configure the Virtual Fabric ID in the cinder.conf file for each Brocade
FC fabric block.


Proposed change
===============

A Brocade FC switch(fixed/modular) can be partitioned into multiple virtual
switches. A unique ID called VFID(Virtual Fabric ID) is assigned to each
virtual switch and all the configurations, including zones, happen within
the context of this VFID. User can configure a particular VFID to be
'default' which will be the default context for when users login to the
switch.

As of Juno, the Brocade Zone Driver and look-up service does not set the
VF context while establishing a session to the switch because of which it
has access only to the zones in the 'default' VFID.

The proposal is to enhance the driver and look-up service to set the VFID
context to support any virtual fabric configured in the chassis.

A new 'fc_virtual_fabric_id' config option is added to zoning fabrics
configuration options as follows:

**fc_virtual_fabric_id** (StrOpt) VFID of virtual fabric. Default value
is 'None'.

The zone driver and look-up service will read the fabric configuration
and extract the VFID to read/apply the configurations on the specific
virtual fabric.

Multiple Virtual Fabrics Support:

Cinder already has support for configuring multiple fabrics. In this
release, we are adding the support to configure multiple **virtual fabrics**.

Configuring multiple virtual fabrics is similar to configuring regular
fabrics. In case of a virtual fabric, the cinder admin will configure
**fc_virtual_fabric_id** config option in addition to all other fabric
configuration options.

On volume attach/detach, the fibre channel volume driver will use SAN lookup
service to traverse through the fabrics configured in cinder.conf to identify
the targets connected to a host and returns a map of initiator and target port
WWNs for each fabric.

Sample map object returned by the look up service::

  {
      <Fabric1>: {
          'initiator_port_wwn_list':
          ('200000051e55a100', '200000051e55a121'..)
          'target_port_wwn_list':
          ('100000051e55a100', '100000051e55a121'..)
      }
      <Virtual_Fabric_2>: {
          'initiator_port_wwn_list':
          ('300000051e55a100', '300000051e55a121'..)
          'target_port_wwn_list':
          ('400000051e55a100', '400000051e55a121'..)
      }
  }

The volume driver will process this information to extract initiator and
target map for each fabric and builds a new initiator_target_map object.

Sample initiator_target_map object::

        {
            'host WWPN 1': ['target WWPN 1', 'target WWPN 2']
            'host WWPN 2': ['target WWPN 3', 'target WWPN 4']
        }

The zone manager will be provided with the above map of initiator and it's
corresponding target port WWNs by the volume driver to create/delete zones.


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

Zoning enforces ACL in SAN. Zoning is same in either regular or virtual
fabrics. There are no specific security impacts with respect to virtual
fabrics.


Notifications impact
--------------------

None.


Other end user impact
---------------------

None.


Performance Impact
------------------

None.


Other deployer impact
---------------------

Configuration of virtual fabric requires providing the **fc_virtual_fabric_id**
config option with valid virtual fabric VF ID in zoning fabric configuration.


Developer impact
----------------

Volume drivers will NOT have to be modified to support virtual fabric.


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

- Enhance the Brocade look-up service to find host and targets connected
  to a VF.
- Enhance the Brocade Zone Driver to execute commands in the context of
  a VFID.
- Unit test the zone driver and look-up service.

Dependencies
============

None.


Testing
=======

Unit tests will be performed to make sure all CRUD operations are
successful on virtual and physical SAN fabrics.

Documentation Impact
====================

Configuration details of **fc_virtual_fabric_id** config option will be added
to the fabric zoning configuration of Brocade Fibre Channel Zone Driver.

References
==========

http://www.brocade.com/downloads/documents/html_product_manuals/FOS_730_CLI/wwhelp/wwhimpl/js/html/wwhelp.htm#href=Title.Fabric_OS.html

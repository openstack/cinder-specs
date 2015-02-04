..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Support Linux on System z (S/390) as a hypervisor platform
================================================================

https://blueprints.launchpad.net/cinder/+spec/linux-systemz

There are some platform-specific changes needed in order to allow Cinder
manage FCP-based block storage on Linux on System z.

Additional OpenStack functionality beyond initial Cinder support is not part of
this blueprint; we will have specific additional blueprints for that, as
needed.

Required support in Nova is described by a separate blueprint which is listed
as a dependency below.

iSCSI does not require any changes.


Problem description
===================

Linux on System z differs from other Linux platforms in the following
respects:

* System z uses a different format for device file paths (ccw-based, rather
  than pci-based)

* Auto-discovery for fibre-channel devices can be configured online, or
  offline. In case auto-discovery is turned off, devices need to be
  added, and removed explicitly by OpenStack (unit_add, unit_remove)

* vHBAs may be turned online, or offline. Offline vHBAs need to be
  ignored.

Proposed change
===============

Change code in Cinder to address these issues, dependent on the host
capabilities indicating a CPU architecture of ``arch.S390X``, and
``arch.S390``.
For details, see section `Work Items`_.

Alternatives
------------

None

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

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None (no need for platform-specific parameters in cinder.conf as part of this
blueprint)

Developer impact
----------------

None (changes should not affect other platforms)

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stefan-amann


Other contributors:
  mzoeller
  maiera


Work Items
----------

In ``cinder/brick/initiator/connector.py``:

* connect_volume needs to support the System z specific format of the
  device file paths and issue the unit_add command (which is done
  by calling the new configure_scsi_device() function)

  ::

    if platform.machine() == 's390x':
        target_lun = "0x00%02d000000000000" %
                     int(connection_properties.get('target_lun', 0))
        host_device = ("/dev/disk/by-path/ccw-%s-zfcp-%s:%s" %
                      (pci_num,
                       target_wwn,
                       target_lun))
        self._linuxfc.configure_scsi_device(pci_num, target_wwn, target_lun)
    else :
        host_device = ("/dev/disk/by-path/pci-%s-fc-%s-lun-%s" %
                      (pci_num,
                       target_wwn,
                       connection_properties.get('target_lun', 0)))
    host_devices.append(host_device)


* disconnect_volume needs to support the System z specific format of the
  device file paths and issue the unit_remove command (which is done by
  calling the new deconfigre_scsi_device() function)

  ::

    for device in devices:
        self._linuxscsi.remove_scsi_device(device["device"])
    if platform.machine() == 's390x':
        ports = connection_properties['target_wwn']
        wwns = []
        # we support a list of wwns or a single wwn
        if isinstance(ports, list):
            for wwn in ports:
                wwns.append(str(wwn))
        elif isinstance(ports, basestring):
            wwns.append(str(ports))
        hbas = self._linuxfc.get_fc_hbas_info()
        for hba in hbas:
            pci_num = self._get_pci_num(hba)
            if pci_num is not None:
                for wwn in wwns:
                    target_wwn = "0x%s" % wwn.lower()
                    target_lun = "0x00%02d000000000000"
                        % int(connection_properties.get('target_lun', 0))
                    host_device = ("/dev/disk/by-path/ccw-%s-zfcp-%s:%s" %
                                  (pci_num,
                                   target_wwn,
                                   target_lun))
                    self._linuxfc.deconfigure_scsi_device(
                                   pci_num,
                                   target_wwn,
                                   target_lun)


In ``cinder/brick/initiator/linuxfc.py``:

* Utility functions to execute the unit_add, or unit_remove command.

  ::

    def configure_scsi_device(self, device_number, target_wwn, lun):
        out = None
        err = None
        zfcp_device_command = ("/sys/bus/ccw/drivers/zfcp/%s/%s/unit_add" %
                                (device_number,
                                 target_wwn))
        try:
            self.echo_scsi_command(zfcp_device_command, lun)
        except putils.ProcessExecutionError as exc:
            LOG.warn(_("zKVM unit_add call failed exit (
                                   %(code)s), stderr (%(stderr)s)")
                     % {'code': exc.exit_code, 'stderr': exc.stderr})



    def deconfigure_scsi_device(self, device_number, target_wwn, lun):
        out = None
        err = None
        zfcp_device_command = ("/sys/bus/ccw/drivers/zfcp/%s/%s/unit_remove" %
                                (device_number,
                                 target_wwn))
        try:
            self.echo_scsi_command(zfcp_device_command, lun)
        except putils.ProcessExecutionError as exc:
            LOG.warn(_("zKVM unit_remove call failed
                        exit (%(code)s), stderr (%(stderr)s)")
                     % {'code': exc.exit_code, 'stderr': exc.stderr})

* Have get_fc_hbas_info() only return enabled vHBAs for System z.

  ::

    hbas_info = []
        for hba in hbas:
            if (platform.machine() != 's390x')
                or (hba['port_state'] == 'Online'):
                    ...same as today

Dependencies
============

Nova blueprint to add support for KVM/libvirt in Linux on System z
https://blueprints.launchpad.net/nova/+spec/libvirt-kvm-systemz


Testing
=======

Unit test:

* Unit tests will be added and performed on System z, as well as
  Intel-based machines.
* We will provide an environment for CI testing on System z.
  This is described by the Nova blueprint which is listed as a dependency.
  We will test Cinder and Nova on System z in this environment.


Documentation Impact
====================

* No changes needed in config docs.

* Doc changes for the platform will be made as needed (details are to be
  determined).


References
==========

* _`[1]` Linux on System z Device Driver book,
  http://public.dhe.ibm.com/software/dw/linux390/docu/l316dd25.pdf

* _`[2]` Linux on System z,
  http://www.ibm.com/developerworks/linux/linux390/

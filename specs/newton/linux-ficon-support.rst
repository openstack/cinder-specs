..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Support ECKD volumes on Linux on System z
================================================================

https://blueprints.launchpad.net/cinder/+spec/linux-ficon-support

FICON (Fibre Connection) is a Fibre-Channel FC-4 layer protocol. It is
used on mainframes to attach ECKD volumes via Fibre-Channel.
Several changes are needed in order to allow Cinder to
manage FICON-based block storage on Linux on System z.

Besides, support is to be added to the existing Cinder drivers
for storage subsystems supporting ECKD (for example, IBM DS8000).

Required support in Nova is described by a separate blueprint which is
listed as a dependency below.


Problem description
===================

FICON and FCP are Fibre-Channel FC-4 layer protocols. FICON uses a
different scheme to address volumes, compared to FCP.

* ECKD volumes are addressed by a Control Unit and Unit Address, rather
  than WWPN and LUN.

* System z uses a different format for device file paths (ccw-based, rather
  than pci-based), similar to the file paths used for FCP on System z

* ECKD volumes need to be set online and formatted before they can be
  used

Use Cases
=========

Allow an end-user to create and manage Cinder block devices based upon
FICON attached storage subsystems supporting the ECKD protocol.
For example: create, delete, image deploy, snapshot, ..

We will extend a Cinder driver for IBMs DS8000. Other vendors, such as
EMC, Hitachi, or HP support ECKD, too. They are potential candidates.

Proposed change
===============

Change code in os-brick to address these issues, dependent on a new
protocol type of 'fibre_channel_eckd', provided by the driver
For details, see section `Work Items`.

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

None

Developer impact
----------------

Platforms that support FICON/ECKD will need to implement this for their
drivers.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stefan-amann


Other contributors:
  arecknag
  maiera


Work Items
----------

In ``os-brick/initiator/connector.py``:

* provide a new connector FibreChannelECKDConnector

* connect_volume needs to enable interrupts for the volume and
  set the volume online. This is done by calling the new
  configure_eckd_device() function)

* disconnect_volume needs to set the volume offline. This is done by
  calling the new deconfigure_eckd_device() function


In ``os-brick/initiator/linuxeckd.py``:

* Utility functions to set the volume online, validate it is formatted
  appropriately, and format it if required

  ::

    def _is_online(self, device):
        """
        Return True if device is online, else return False
        :param device: device id
        """
        if os.access('/sys/bus/ccw/devices/%s/online' % device, os.R_OK):
            online_file = None
            try:
                online_file = open('/sys/bus/ccw/devices/%s/online' % device)
                value = online_file.readline()
                if value and value.strip() == '1':
                    return True
            finally:
                if online_file:
                    online_file.close()
        return False

    def _is_formatted(self, device):
        """
        Return True if device is online, else return False
        :param device: device id
        """
        if os.access('/sys/bus/ccw/devices/%s/status' % device, os.R_OK):
            formatted = None
            (out, _err) = self._execute('cat',
                                '/sys/bus/ccw/devices/%s/status' % device,
                                run_as_root=True,
                                root_helper=self._root_helper)
            if out and out.strip() == 'unformatted':
                return False
        return True

    def format_eckd_volume(self, path):
        """
        formats ECKD volume
        :param device: device path
        """
        name = os.path.realpath(path)
        if name.startswith("/dev/"):
            format_cmd = 'dasdfmt -y -b 4096 -d ldl ' + name
            (out, _err) = self._execute(format_cmd, run_as_root=True,
                                root_helper=self._root_helper)
            return True
        else:
            return False


In ``os-brick/initiator/linuxfc.py``:

* new class LinuxFibreChannelECKD

.. code-block:: python

    def configure_eckd_device(self, device_number):
        """Add the eckd volume to the Linux configuration. """

        full_device_identifier = "0.0.%04x" % device_number
        eckd_device_command = ('cio_ignore', '-r',
            '%(dev_id)s' % {"dev_id": full_device_identifier})
        LOG.debug("issue cio_ignore for s390: %s", eckd_device_command)

        out, info = None, None
        try:
            out, info = self._execute('cio_ignore', '-r',
                    '%(dev_id)s' % {"dev_id": full_device_identifier},
                    run_as_root=True,
                    root_helper=self._root_helper)
        except putils.ProcessExecutionError as exc:
            LOG.warning(_LW("cio_ignore call for s390 failed exit"
                            " %(code)s, stderr %(stderr)s"),
                        {'code': exc.exit_code, 'stderr': exc.stderr})

        eckd_device_command = ('chccwdev', '-e',
                '%(dev_id)s' % {"dev_id": full_device_identifier})
        LOG.debug("add ECKD command for s390: %s", eckd_device_command)
        out, info = None, None
        try:
            out, info = self._execute('chccwdev', '-e',
                    '%(dev_id)s' % {"dev_id": full_device_identifier},
                    run_as_root=True,
                    root_helper=self._root_helper)
        except putils.ProcessExecutionError as exc:
            LOG.warning(_LW("add ECKD call for s390 failed exit"
                            " %(code)s, stderr %(stderr)s"),
                        {'code': exc.exit_code, 'stderr': exc.stderr})

    def deconfigure_eckd_device(self, device_number):
        """Remove the eckd volume from the Linux configuration. """

        LOG.debug("Deconfigure ECKD volume: device_number=%(device_num)s ",
                  {'device_num': device_number})

        full_device_identifier = "0.0.%04x" % device_number
        eckd_device_command = ('chccwdev', '-d',
                '%(dev_id)s' % {"dev_id": full_device_identifier})
        LOG.debug("remove ECKD command for s390: %s", eckd_device_command)
        out, info = None, None
        try:
            out, info = self._execute('chccwdev', '-d',
                    '%(dev_id)s' % {"dev_id": full_device_identifier},
                    run_as_root=True,
                    root_helper=self._root_helper)
        except putils.ProcessExecutionError as exc:
            LOG.warning(_LW("remove ECKD call for s390 failed exit"
                            " %(code)s, stderr %(stderr)s"),
                        {'code': exc.exit_code, 'stderr': exc.stderr})

Adaptations to volume.filters to allow the new commands to be sent


Cinder drivers need to report the Control Unit address, and Unit Address
of the ECKD volume and set the driver_volume_type to 'fibre_channel_eckd'

Any drivers wishing to support FICON will need to have CI reporting results
from running against a FICON attached host.

Dependencies
============

Nova blueprint to add support for ECKD for Linux on System z. The blueprint
can be found here:
https://blueprints.launchpad.net/nova/+spec/linux-ficon-support

Testing
=======

Unit test:

* Unit tests will be added and performed on System z, as well as
  Intel-based machines.

CI environment:

* We will provide a 3rd party CI environment for DS8000.


Documentation Impact
====================

* We will update documentation as required.


References
==========

* _`[1]` Linux on System z Device Driver book,
  http://public.dhe.ibm.com/software/dw/linux390/docu/l316dd25.pdf

* _`[2]` Linux on System z,
  http://www.ibm.com/developerworks/linux/linux390/


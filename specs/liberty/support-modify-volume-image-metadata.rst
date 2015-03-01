..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Support Modifying Volume Image Metadata
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/python-cinderclient/+spec/support-modify-volumn-image-metadata

This blueprint intends to support modifying volume image metadata, provide
Cinder API to allow a user to modify an image property or add new properties.

Cinder should provide the similar mechanism for protected properties as what
Glance did, and it is the agreed approach according to the discussion in the
IRC and mailing list.

Problem description
===================

When creating a bootable volume from an image, the image metadata (properties)
is copied into a volume property named volume_image_metadata.

Cinder volume_image_metadata is the metadata on bootable volumes that Nova
looks at and uses for scheduling as well as for passing along to the individual
compute drivers.

Cinder volume_image_metadata is used by nova for things like scheduling and
for setting device driver options. nova treat volume_image_metadata the same
way as the properties on images. This information may need to change after
the volume has been created from an image, besides, the additional properties
may also needed to make it available in the scheduler (detailed in the below
sections). So, There should be a way to support change/update image metadata.

Use Cases
=========

Here are some types of metadata properties that if set will affect runtime
characteristics of how Nova handles the booted volume. Many of them very
well could be a user deciding to basically build a new image using a volume,
but they want to twiddle with various handling properties as part of the
building process. Or they simply may have a bootable volume that they want
to modify some of these properties.

Hypervisor selection: OpenStack Compute supports many hypervisors, although
most installations use only one hypervisor. For installations with multiple
supported hypervisors, you can schedule different hypervisors using the
ImagePropertiesFilter. This filters compute nodes that satisfy any
architecture, hypervisor type, or virtual machine mode properties specified
on the instance's image properties (also volume_image_properties).

Virtual CPU Topology: This provides the preferred socket/core/thread counts
for the virtual CPU instance exposed to guests. This enables the ability to
avoid hitting limitations on vCPU topologies that OS vendors place on their
products. See also:
`<http://git.openstack.org/cgit/openstack/nova-specs/tree/specs/juno
/virt-driver-vcpu-topology.rst>`_

Watchdog Behavior: For the libvirt driver, you can enable and set the behavior
of a virtual hardware watchdog device for each flavor. Watchdog devices keep
an eye on the guest server, and carry out the configured action, if the server
hangs. The watchdog uses the i6300esb device (emulating a PCI Intel 6300ESB).
If hw_watchdog_action is not specified, the watchdog is disabled. Watchdog
behavior set using a specific image's properties will override behavior
set using flavors.

Shutdown Behavior: Numerous things coming:
https://review.openstack.org/#/c/89650/12 What landed in Juno: By default,
guests will be given 60 seconds to perform a graceful shutdown. After that,
the VM is powered off. The os_shutdown_timeout property allows overriding
the amount of time (unit: seconds) to allow a guest OS to cleanly shut down
before power off. A value of 0 (zero) means the guest will be powered off
immediately with no opportunity for guest OS clean-up.

Minimum Flavor Requirements: Minimum required CPU, Minimum RAM. These are
used by the Horizon UI to guide flavor selection. Somebody with a bootable
volume may install some new software on it and play around with setting
the minimum requirements before cloning to an image.

The new Horizon UI is going to be providing more information about images,
flavors, volumes to admins and users in the UI leveraging the metadata and
looking up rich information about the metadata from the definition catalog
to display information to users and admins. This can include metadata about
software on the volume.

Proposed change
===============

This ONLY affects the individual volume, it's has nothing to do with volume
types

We are proposing the changes in Cinder to add update capability and provide
new Cinder API to allow a user to update an image properties.

Since Glance has RBAC (role based access control) on some specific
properties, the RBAC config file is proposed to copied from Glance to Cinder,
and sync the protected properties code from Glance into Cinder (Glance
are happy to take code cleanups that make this easier; we can consider
OSLO or whatever in future but that's a heavy weight process for another
day, copy and paste will do for now).


Alternatives
------------

As to the strategy on protected properties, one proposal is providing new API
to query the properties protected or so per-property from Glance to Cinder,
but his approach seems will increase the overload of Glance.

Data model impact
-----------------

None

REST API impact
---------------

Since only image metadata is used by nova for VM scheduling or setting
device driver options, we proposed to add new REST APIs into Cinder for
the operations on image metadata of volume.

* update image metadata referenced with volume


**Common http response code(s)**

* Modify Success: `200 OK`
* Failure: `400 Bad Request` with details.
* Forbidden: `403 Forbidden`       e.g. no permission to update the
                                        specific metadata
* Not found: `501 Not Implemented` e.g. The server doesn't recognize
                                        the request method
* Not found: `405 Not allowed` e.g. HEAD is not allowed on the resource


**Update volume image metadata**
  * Method type
    PUT

  * API version
    PUT /v2/{project_id}/volumes/{volume_id}/image_metadata

  * JSON schema definition
    {
       "image_metadata": {
            "key": "v2"
        }
    }

    To unset a image metadata key value, specify only the key name.
    To set a image metadata key value, specify the key and value pair.


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* We intend to expose this via Horizon and are working on related blueprints.
* Glance also need share its properties protection code to Cinder
  and some code cleanups in Glance's IMPL.
* Provide Cinder API to allow a user to update an image property.
  CLI-python API that triggers the update.

  # Sets or deletes volume image metadata
  cinder image-metadata  <volume-id> set <property-name = value>

Performance Impact
------------------

None anticipated.

Other deployer impact
---------------------

* Two config file will be added into Cinder, that is property-protections-
  policies.conf and property-protections-roles.conf
  These file will be put in "/etc/cinder" by default and is configurable via
  cinder.conf or point directly at the Glance files in devstack for example.
* Deployer will be responsible for keeping the config files
  in sync with Glance's
* The config files will only take effect when they are present on the system.
  So it is up to the deployer to ensure they are accurate. Otherwise, there
  will be no impact to Cinder of the OpenStack environment by default.

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
 Dave Chen (wei-d-chen)

Other contributors:
 None


Work Items
----------

Changes to Cinder:

#. Define property protections config files in Cinder
   (Deployer need to keep the files in sync with Glance's)
#. Sync the properties protection code from Glance into Cinder
   (The common protection code will be shared in Cinder)
#. Extend existing volume_image_metadatas(VolumeImageMetadataController)
   controller extension to add update capability.
#. Reuse update_volume_metadata method in volume API for updating image
   metadata and differentiate user/image metadata by introducing a new
   constant "meta_type"
#. Add update_volume_image_metadata method to volume API.
#. Check against property protections config files
   (property-protections-policies.conf or property-protections-roles.conf)
   if the property has update protection.
#. Update DB API and driver to allow image metadata updates.

Changes to Cinder python client:

#. Provide Cinder API to allow a user to update an image property.
   CLI-python API that triggers the update.
   # Sets or deletes volume image metadata
   cinder image-metadata  <volume-id> set <property-name = value>

Dependencies
============

Same dependencies as Glance.

Testing
=======

Unit tests will be added for all possible code with a goal of being able to
isolate functionality as much as possible.

Tempest tests will be added wherever possible.


Documentation Impact
====================

Since Glance has role based access control to properties. It could be the case
that we want to update a property in Cinder that is protected in Glance.
Eg: a license key is added in glance and it's copied to cinder when the volume
is created. It should not be changed by an unauthorized user in Cinder because
this can be violating the billing policies for that image. Therefore, Property
Protections which is similar with Glance is proposed to be adopted into Cinder.

We propose to define two samples config file in favor of Property Protections,
that is property-protections-roles.conf and property-protections-policies.conf.

* property-protections-policies.conf
This is a template file when using policy rule for property protections.

Example: Limit all property interactions to admin only using policy
rule context_is_admin defined in policy.json.

+-------------------------------------------------------------------+
| [.*]                                                              |
+===================================================================+
| create = context_is_admin                                         |
+-------------------------------------------------------------------+
| read = context_is_admin                                           |
+-------------------------------------------------------------------+
| update = context_is_admin                                         |
+-------------------------------------------------------------------+
| delete = context_is_admin                                         |
+-------------------------------------------------------------------+

* property-protections-roles.conf
This is a template file when property protections is based on user's role.
Example: Allow both admins and users with the billing role to read and modify
properties prefixed with x_billing_code_.

+-------------------------------------------------------------------+
| [^x_billing_code_.*]                                              |
+===================================================================+
| create = admin,billing                                            |
+-------------------------------------------------------------------+
| read = admin, billing                                             |
+-------------------------------------------------------------------+
| update = admin,billing                                            |
+-------------------------------------------------------------------+
| delete = admin,billing                                            |
+-------------------------------------------------------------------+

Please refer to here, http://docs.openstack.org/developer/glance/property-protections.html
for the details explanation of the format.

In case there is property which is protected strictly in Glance, license key
for example, deployer should aware the config files may turn out to be
inconsistent between Cinder and Glance, it's up to deployer's responsibility
to keep the config files in sync with Glance's

Other docs is also needed for new API extension and usage.

References
==========

This blueprint is actually a partial task of Graffiti project, many
parts of this concept have already been implemented for other pieces
of OpenStack, but that Cinder is outstanding (already completed for
images, flavors, host aggregates)

`Youtube summit recap of Graffiti Juno POC demo.
<https://www.youtube.com/watch?v=Dhrthnq1bnw>`_

`Discussions in the mailing list.
<http://openstack.10931.n7.nabble.com/cinder-glance-Update-volume
-image-metadata-proposal-tt44371.html#a44523>`_

`Discussions in the IRC.
<http://eavesdrop.openstack.org
/meetings/glance/2014/glance.2014-06-26-20.03.log.html>`_

`The Horizon patch set which depends on this functionality
<https://review.openstack.org/#/c/112880/>`_

`Property Protections introduction in Glance
<http://docs.openstack.org/developer/glance/property-protections.html>`_

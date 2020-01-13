..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Support Glance multiple stores
==============================

https://blueprints.launchpad.net/cinder/+spec/support-glance-multiple-backend

This blueprint proposed to support Glance multiple stores.


Problem description
===================

In Rocky and Stein, Glance has added the ability to configure multiple stores
as an EXPERIMENTAL feature [1]_. This feature is on schedule to be fully
supported in the Train cycle. This way an operator can configure more than one
of similar or different kind of stores and use one as a default store. If a
store is not specified at the time of uploading an image then the image will be
stored in default store.

In case of Cinder upload-volume-to-image, if no changes are made to Cinder,
even if multiple stores are configured, the volume image will always be
uploaded to the default store. This does not cause any issues, but does not
allow an operator to take advantage of Glance multiple stores.

Use Cases
=========

1. An operator is offering multiple Glance image stores distinguished by some
   desirability metric (as explained to end users in each store's description)
   and wants a user to have the option to select which of these stores will be
   used to store an image created from a Cinder volume.

Proposed change
===============

Define a field named 'image_service:store_id' in the volume-type extra-specs.
Using 'image_service' as the namespace for the 'store_id' key signals to the
scheduler that this field should be ignored when scheduling a volume build.

The value is the store 'id' as indicated in the Image Service API v2 stores
discovery response::

    GET $IMAGE_API_URL/v2/info/stores

    {
        "stores": [
            {
                "id":"reliable",
                "description": "Has a good transfer speed/price ratio"
            },
            {
                "id":"fast",
                "description": "Fast store with short transfer time",
                "default": true
            },
            {
                "id":"cheap",
                "description": "Inexpensive store for seldom used images"
            }
        ]
    }

There was discussion at the Train Forum and PTG concerning the validation of
extra_specs [2]_, so the question arises of how that would impact the proposed
usage of extra_specs for the Glance store information.  If the validation
implementation follows the Kilo spec [3]_, validation will happen in the REST
API.  The validation code could filter out the 'image_service:store_id' field
before validating the other fields with the appropriate Cinder backend driver.
Additionally, the code could validate the 'image_service:store_id' setting by
making the Image Service API stores-info call and verfiying that value occurs
as an element of the list given by the JSONPath ``$.stores[*].id`` in the
response.  (This would be ``["reliable", "fast", "cheap"]`` in the example
above.)

An objection to this proposal is that the Glance store should be associated
directly with the image-type, and not hidden in with the extra-specs.
Consider, however, how this would look to end users.  If the new field is made
part of the volume_type, it would be handled like the 'qos_specs_id', and the
type-show response would look something like this::

    {
        "volume_type": {
            "description": "A new type",
            "extra_specs": {},
            "id": "9f319ace-a53c-40cb-8171-f12c4547baaf",
            "is_public": true,
            "name": "a_new_type",
            "os-volume-type-access:is_public": true,
            "qos_specs_id": null,
            "image_service:store_id": null
        }
    }

A null value here means that the default Glance image store will be used, but
this is confusing to end users because a natural interpretation of the null
value is that this volume_type does not allow images to be uploaded to Glance.
If we use the extra_specs, however, a volume type using the default Glance
image store would look like::

    {
        "volume_type": {
            "description": "Volume type using Glance default store",
            "extra_specs": {
                "volume_backend_name": "lvmdriver-1"
            },
            "id": "686cc9f0-f4ca-4a5a-b113-eeb57d3977fd",
            "is_public": true,
            "name": "lvmdriver-1",
            "os-volume-type-access:is_public": true,
            "qos_specs_id": null
        }
    }

Which is exactly the same as we have currently, while a volume type that
wants to specify a store would look like this::

    {
        "volume_type": {
            "description": "Volume type specifying a Glance store",
            "extra_specs": {
                "volume_backend_name": "lvmdriver-1",
                "image_service:store_id": "cheap"
            },
            "id": "686cc9f0-f4ca-4a5a-b113-eeb57d3977fd",
            "is_public": true,
            "name": "lvmdriver-1",
            "os-volume-type-access:is_public": true,
            "qos_specs_id": null
        }
    }

Which, as you'd expect, only brings the Glance store ID to your attention if
something different than the usual case is being specified.

Additionally, this proposal would require no changes to the Cinder data
model or the REST API, though these considerations are secondary to the
usability implications discussed above.

Vendor-specific changes
-----------------------
None

Alternatives
------------

* Make the 'image_service:store_id' a property of the volume_type instead of
  an extra_specs field.  While this would respect the intention of extra_specs
  describing properties of the Cinder backend, it has the usability issues
  mentioned in the `Proposed change`_ section and would require a new API
  microversion and a database change.

* Add a new Cinder configuration option 'store' under the 'glance' section to
  upload all the volume images to a specific glance store. If this option is
  not present then all the volume images will be uploaded to default glance
  store. This solution will be efficient if operator doesn't want to expose the
  use of uploading volume image to specific store to end user. This will be a
  very simple change and doesn't require any Block Storage API change or change
  in python-cinderclient.

  This alternative has the downside that all images of Cinder volumes must be
  stored in the same store.  If an operator has taken the trouble to configure
  multiple stores in Glance, presumably there is end-user demand to use
  different stores for different purposes, and presumably such an operator will
  want block storage users to be able to take advantage of the Glance multiple
  stores feature.

* Add a new microversion to the volume-upload-to-image API to support uploading
  a volume image to a specific glance store of an end user's choice.  This
  could be done by adding a new input request parameter 'store' to the
  volume-upload-to-image API.  If the 'store' option is not specified in a
  request then the image will be uploaded to the default store.  This has the
  advantage that an end user will be able to choose any of the available Glance
  stores.

  This alternative has the disadvantage that the end user will have to make the
  ``GET /v2/info/stores`` call to the Image API to discover the store
  identifiers in use in a particular cloud.  It also has the disadvantage that
  it works differently from Cinder's support of multiple back ends, where the
  backend a volume is stored in is determined by the volume type.  It would be
  better to use a workflow more familiar to Cinder users.

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

None.  User experience will be consistent with current practice, where an end
user selects a volume type based on specific features exposed by the cloud
operator.

Performance Impact
------------------

None


Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  abhishek-kekane

Other contributors:
  brian-rosmaita

Work Items
----------

* Add 'image_service:store_id' handling to the Cinder create-image-from-volume
  workflow
* Add related tests


Dependencies
============

None


Testing
=======

* Add related unittest
* Add related functional test
* Add tempest tests


Documentation Impact
====================

Operators documentation should be updated according to spec implementation.


References
==========

.. [1] https://docs.openstack.org/glance/rocky/admin/multistores.html
.. [2] https://etherpad.openstack.org/p/denver-forum-cinder-improving-drvr-cap-rep
.. [3] https://review.opendev.org/#/c/131280/

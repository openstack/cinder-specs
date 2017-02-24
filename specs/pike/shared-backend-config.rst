..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Shared backend config stanza
==========================================

https://blueprints.launchpad.net/cinder/+spec/shared-backend-config

Add a new config section for shared backend configuration options.

Problem description
===================

Right now there isn't a great way to share config options between backends
in a multi-backend cinder.conf. You can put things in DEFAULT, but drivers
self.configuration won't be able to see it if they were defined in their own
stanza. You can just look directly at the DEFAULT section, but you then can't
easily override it in the driver backend section if you *did* want to have
something special. As an example consider the following cinder.conf::

    [DEFAULT]
    enabled_backends = backend1, backend2, backend3
    ...
    my_driver_param = foo
    ...

    [backend1]
    ...
    my_driver_param = bar

    [backend2]
    ..
    # not specifying the param

    [backend3]
    ...
    (also not specifying, maybe even a different volume driver, doesn't matter)

What we end up with is backend1 gets 'bar' and backend2/3 get whatever the
default was for 'my_driver_param'. To get them to use 'foo' we need something
like::

    [DEFAULT]
    enabled_backends = backend1, backend2, backend3
    ...

    [backend1]
    ...
    my_driver_param = bar

    [backend2]
    ..
    my_driver_param = foo

    [backend3]
    ...
    my_driver_param = foo

Part of the confusion is that if you don't use the multi-backend format you
actually do want the options set in DEFAULT.

Use Cases
=========

The main use-case here is for shared configs relevent to multiple backends
in cinder.conf. Also to help lessen some of the confusion about shared config
options in DEFAULT.

Proposed change
===============

The new config section called 'backend_defaults' will contain config options
that each backend has access to. If any options are also in the backend stanza
that backend specific value will take precedence. It will be safe to assume
that config options in shared will be shared and available for all backends.
None of the guesswork from DEFAULT.

A new config then might look something like::

    [DEFAULT]
    enabled_backends = backend1, backend2, backend3
    ...

    [backend_defaults]
    my_driver_param = foo

    [backend1]
    ...
    my_driver_param = bar

    [backend2]
    ..
    # not defining the value, but it gets picked up

    [backend3]
    ...
    # same here..

This is the same as the example above where all backends define the value. This
simplistic example doesn't really show a ton of the benefits, so a slightly
more realistic example might be::


    [DEFAULT]
    enabled_backends = pure1, pure2, pure3
    ...

    [backend_defaults]
    volume_driver = cinder.volume.drivers.pure.PureISCSIDriver
    driver_ssl_verify = true
    driver_ssl_cert = /etc/blah/my_cert
    use_multipath_for_image_xfer = true
    use_chap_auth = true
    image_volume_cache_enabled = true
    image_volume_cache_max_count = 15
    image_volume_cache_max_size_gb = 200
    max_over_subscription_ratio = 10
    pure_eradicate_on_delete = false

    [pure1]
    san_ip = 1.2.3.4
    pure_api_token = abcdef123456
    max_over_subscription_ratio = 50

    [pure2]
    san_ip = 1.2.3.5
    pure_api_token = abcdef1234567
    image_volume_cache_max_count = 25

    [pure3]
    san_ip = 1.2.3.6
    pure_api_token = abcdef12345678

So here we have a ton of shared config options, leaving only a few specific to
each backend.

In addition to the shared config section we will deprecate setting backends
via the DEFAULT section. We will only allow multi-backend style configs using
the "enabled_backends" config option.

Alternatives
------------

We could change the behavior so that things in DEFAULT do this, but it not
be backwards compatible with older cinder.confs. Behavior would probably not
be what deployers would expect when upgrading.

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

Would have a new config mechanism to learn, but older style configs would still
work the same way. They will eventually need to migrate to using the
"enabled_backends" config option, this process will follow the standard
deprecation process.

Developer impact
----------------

Nothing other than needing to be aware of how drivers can get their config
values.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  patrick-east

Work Items
----------

* Add a warning when initializing c-vol without "enabled_backends"
* Modify the driver config utility to look for this section first, and override
values with backend specific ones if they are defined.
* Add some unit tests
* Incorporate into multi-backend tempest test configuration

Dependencies
============

None


Testing
=======

* Unit tests
* Tempest test config with devstack (reach goal, depends on timing)


Documentation Impact
====================

* Updated driver config references
* Driver dev documentation to explain how this works


References
==========

* https://blueprints.launchpad.net/cinder/+spec/shared-backend-config

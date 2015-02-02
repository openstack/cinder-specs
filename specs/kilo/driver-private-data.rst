..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add DB table for driver private data
==========================================

https://blueprints.launchpad.net/cinder/+spec/driver-private-data

As discussed at the mid-cycle meetup (https://etherpad.openstack.org/p/cinder-
meetup-winter-2015) we want to add a DB table where drivers can store info they
need to operate. A specific use case is for iSCSI CHAP credentials for
initiators, but it should be generic enough for other drivers to store data
they depend on too.


Problem description
===================

Currently there is not a standard solution for drivers to store information
they need to operate. For things volume related we have some "provider"
fields on the volume table that can be used, but for things not associated
with the volume's life-cycle they are insufficient. This has lead to some
drivers storing information on the backend which works for some but not all
backends have this capability.

A use case for this is storing target CHAP secrets so iSCSI initiators can
do CHAP authentication against targets.


Proposed change
===============

A solution for this problem is to have a new table in the Cinder database which
can hold initiator information for drivers. The model would look like:

::

  +--------------+--------------+--------------------------------------------+
  |   Field      |     Type     |            Description                     |
  +--------------+--------------+--------------------------------------------+
  | created_at   | datetime     |                                            |
  | updated_at   | datetime     |                                            |
  | id           | int(11)      | Auto incremented ID                        |
  | initiator    | varchar(255) | Unique constraint with "key" +             |
  |              |              | "namespace"                                |
  | namespace    | varchar(255) | Unique constraint with "key" + "initiator" |
  |              |              | This comes from cinder.conf for each back  |
  |              |              | end configuration.                         |
  | key          | varchar(255) | Unique constraint with "initiator" +       |
  |              |              | "namespace"                                |
  | value        | varchar(255) | This could be anything a driver wants to   |
  |              |              | keep track of for this initiator. In the   |
  |              |              | CHAP credential usecase this could be a    |
  |              |              | barbican key uuid.                         |
  +--------------+--------------+--------------------------------------------+

In initialize_connection for VolumeManager if there is an initiator (in the
case of iSCSI) we can look up the rows for it filtering by the namespace
and pass them in to the driver as a new optional parameter for
initialize_connection.

The namespace field will come from a new configuration option on the driver
base class called driver_data_namespace. This config option will default to
None. If this option is not set, we will check the backend_name config option,
and finally if that is not set, we fall back to the driver class name. This
will allow for being able to share data between multiple different backends
and a single backend in HA configuration.

The driver then can return a model update dictionary as an "initiator_update"
field on the connection info. These changes will then be applied to the
initiator table.

The initiator_update dictionary can contain "set_values" dictionary and/or
"remove_values" list. The set_values are key value pairs which are to
be set or updated in the database. The remove_values is a list of keys to be
deleted from the database. They will be "hard" deleted so the data will not
remain in the database once specified in remove_values.

The return value from initialize_connection might look somthing like:

::

  {
      "driver_volume_type": "iscsi",
      "data": {
          "target_iqn": "iqn.2010-06.com.purestorage:flasharray.12345abc",
          "target_portal": 1.2.3.4:5678,
          "target_lun": 1,
          "target_discovered": True,
          "access_mode": "rw",
          "auth_method": "CHAP",
          "auth_username": "someUserName",
          "auth_password": "somePassword"
      },
      "initiator_updates":
          {
              "set_values": {
                  "purearray1ChapSecretId": "someUUID",
                  "someOtherThing": "someOtherValue"
              }
              "remove_values": [
                  "myOldKey",
                  "anotherKey"
              ]
          }
  }

The manager's initialize_connection method can then pull the updates
out of the return object, do the database operations (as needed) and
pass on the connection info with the field stripped out so it does not
propagate through the system.

Alternatives
------------

* These key values pairs are scoped for the initiator, so there is a chance
  for two different backends to get access to the values. This is helpful for
  backends that are configured in active/active HA, but there is a risk of two
  different vendors being used by the same initiator and having key
  collisions. We could introduce a vendor-prefix field to use as part of the
  key as well to avoid that.

  * We could also use the cinder host to scope these instead of a vendor
    prefix

* We could make drivers keep track of this outside of Cinder. This could be
  left up to a driver backend to store data, or with another external database
  somewhere. Some drawbacks to this approach include having different
  databases to maintain, update, and keep in sync which would be a less than
  desireable user experience for a cloud admin, having multiple ways of doing
  synchronization/locking in drivers can and will introduce bugs and
  maintenance problems, and makes it more difficult for new driver developers
  to make a good choice on where to store data.

* We could allow the drivers direct(ish) access to the database through some
  get/set api's to store arbitrary key-value pairs. Some alternatives for the
  database table in that model include:

  * Have a column for the host name and using it as part of the unique
    constraint for the "key". The benefit being that it is much less likely to
    have naming collisions between drivers using this. The downside is that two
    hosts using the same driver may not be able to easily share data, which
    makes active/active HA for the same backend difficult.

  * Have a column for the driver class name and using it as part of the unique
    constraint for the "key". Again the benefit being to help avoid collisions
    with key fields. This one is better for sharing between hosts that are
    using the same driver type, but the issue then is where does the name come
    from? If it is reported by the driver as a string it might as well be any
    generic prefix as there is nothing to stop someone from just making up
    their name. If it is automatically gathered from the driver class via
    python internal variables on the object then there are issues with any
    driver that changes their class name later on.

  * Have a column that is just a provider/vendor namespace and is part of the
    unique constraint for the "key". Yet again the goal is to avoid collisions
    with key fields. Similar to the last one the only downside is where it
    comes from and that there isn't really anything that would prevent drivers
    from using the same one, but it would be one more layer of protection.

  * Have no restriction and let the keys be more like a global scope, this
    allows drivers to share data easily, but has issues with collisions.

* Return a tuple instead of adding the data to the return object of
  initialize_connection. This requires having an optional second return
  value and conditional calling/return values for methods, or modifying
  every usage of driver.initialize_connection(...).


Data model impact
-----------------

The proposed solution will be adding in a new database table named
driver_initiator_data and associated model DriverInitiatorData. There will be
a migration to add and remove the table but will not seed or modify any data.

Access to the database will be used through api's outlined in the Proposed
Change.

Table rows will be hard deleted, so we don't keep around potentially sensitive
credentials/information.

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

Potential impact for drivers that heavily utilize this table with extra
database queries.

Other deployer impact
---------------------

New config option to potentially set for each backend.

Developer impact
----------------

Drivers that store data on their backend may want to utilize this feature.

This change will require modifying the signature for all drivers
initialize_connection method implementations.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  patrick-east

Other contributors:
  None

Interested parties:
  walter-boring

Work Items
----------

* Add new Model

* Add new db API's

* Add new db Migration scripts

* Modify the volume manager's initialize_connection method to query/save data

Dependencies
============

None

Testing
=======

* DB migration tests

* DB API tests

* BaseDriver unit tests

* No new tempest tests


Documentation Impact
====================

None

References
==========

* https://etherpad.openstack.org/p/cinder-meetup-winter-2015

* https://review.openstack.org/#/c/151837/

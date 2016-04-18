..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Add Generic Volume Group Into Quota Management
==============================================

https://blueprints.launchpad.net/cinder/+spec/add-volumegroup-into-quota-management

Generic Volume Group currently has its own quota mechanism. But we can't get
any information about the Generic Volume Group quota at the API layer.

Problem description
===================

Cinder already achieved Generic Volume Group quota mechanism. It means that
there is a Generic Volume Group quota class(the hard_limit is 10) at the DB
layer. But the Generic Volume Group's quota information wasn't contained in any
quota API's response body, so that users can't get or update it.

Use Cases
=========

Cinder should allow users to get and update the Generic Volume Group's quota.
It's terrible that users could only use the Generic Volume Group quota's
default value and can't update it. With this change, users could 1) change the
value of Generic Volume Group's hard limit, 2) get the Generic Volume Group's
quota usage, through the Cinder's quota API.

Proposed change
===============

Let quota management undertake Generic Volume Group quota. And these five APIs
will be changed:
1) quota-class-show
::

    GET /os-quota-class-sets/default

Add a new line in the response body.
::

    {
        "quota_class_set": {
            "groups": 10
        }
    }

2) quota-show, quota-usage
::

    GET /os-quota-sets/{project_id}?usage={False, True}

Add a new line in the response body.
::

    {
        "quota_set": {
            "groups": {
                "reserved": 0,
                "allocated": 0,
                "limit": 10,
                "in_use": 0
            }
        }
    }

3) quota-defaults
::

    GET /os-quota-sets/{project_id}/defaults

Add a new line in the response body.
::

    {
        "quota_set": {
            "groups": 10
        }
    }

4) quota-class-update
::

    PUT /os-quota-class-sets/default

Allow update "groups" and Add a new line in the response body.
::

    {
        "quota_class_set": {
            "groups": 10
        }
    }

5) quota-update
::

    PUT /os-quota-sets/{project_id}

Allow update "groups" and Add a new line in the response body.
::

    {
        "quota_set": {
            "groups": 10
        }
    }


Alternatives
------------

Leave as is.

Data model impact
-----------------

None

REST API impact
---------------

- The response body of "quota-defaults", "quota-usage", "quota-show" and
  "quota-class-show" APIs will contain Generic Volume Group.
- The "quota-update" and "quota-class-update" APIs will accept the
  "groups" parameter and the response body will contain Generic Volume Group.

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

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  wangxiyuan

Work Items
----------

- Add Generic Volume Group's quota to quota APIs
- Add and update the unit tests
- Update the CinderClient's quota commands


Dependencies
============

None

Testing
=======

Standard unit tests and manual testing.

Documentation Impact
====================

- The response body of quota-defaults, quota-usage, quota-show and
  quota-class-show should be updated.
- The request body of quota-update, quota-class-update should be updated.

References
==========

None

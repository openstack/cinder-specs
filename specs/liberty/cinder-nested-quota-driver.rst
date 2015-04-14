..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Cinder Nested Quota Driver
=================================================================

`bp cinder-nested-quota-driver
<https://blueprints.launchpad.net/cinder/+spec/cinder-nested-quota-driver>`_

Nested quota driver will enable OpenStack to enforce quota in nested
projects.

`bp hierarchical-multitenancy
<https://blueprints.launchpad.net/keystone/+spec/hierarchical-multitenancy>`_


Problem description
===================

OpenStack is moving towards support for hierarchical ownership of projects.
In this regard, Keystone will change the organizational structure of
OpenStack, creating nested projects.

The existing Quota Driver in Cinder called ``DbQuotaDriver`` is useful to enforce
quotas at both the project and the project-user level provided that all the
projects are at the same level (i.e. hierarchy level cannot be greater
than 1).

The proposal is to develop a new Quota Driver called ``NestedQuotaDriver``,
by extending the existing ``DbQuotaDriver`` which will allow enforcing quotas
in nested projects in OpenStack. The nested projects are having a hierarchical
structure, where each project may contain users and projects (can be called
sub-projects).

Users can have different roles inside each project: A normal user can make
use of resources of a project. A project-admin, for example can be a user
who in addition is allowed to create sub-projects, assign quota on resources
to these sub-projects and assign the sub-project admin role to individual users
of the sub-projects. Resource quotas of the root project can only be set by the
admin of the root project. The user roles can be set as inherited, and if set,
then an admin of a project is automatically an admin of all the projects in the
tree below.


Use Cases
---------

**Actors**

* Mike - Cloud Admin (i.e. role:cloud-admin) of ProductionIT
* Jay - Manager (i.e. role: project-admin) of Project CMS
* John - Manager (i.e. role: project-admin) of Project ATLAS
* Eric - Manager (i.e. role: project-admin) of Project Operations
* Xing - Manager (i.e. role: project-admin) of Project Services
* Walter - Manager (i.e. role: project-admin) of Project Computing
* Duncan - Manager (i.e. role: project-admin) of Project Visualisation

The nested structure of the projects is as follows.

.. code:: javascript

     {
        ProductionIT: {
                       CMS  : {
                                 Computing,
                                 Visualisation
                             },
                       ATLAS: {
                                 Operations,
                                 Services
                           }
                    }
      }

Mike is an infrastructure provider and offers cloud services to Jay for
Project CMS, and John for Project ATLAS. CMS has two sub projects below it
named, Visualisation and Computing, managed by Duncan and Walter respectively.
ATLAS has two sub projects called Services and Operations, managed by
Xing and Eric respectively.

1. Mike needs to be able to set the quotas for both CMS and ATLAS, and also
   manage quotas across the entire projects including the root project,
   ProductionIT.
2. Jay should be able to set and update the quota of Visualisation and Computing.
3. Jay should be able to able to view the quota of CMS, Visualisation and
   Computing.
4. Jay should not be able to update the quota of CMS, although he is the
   Manager of it. Only Mike can do that.
5. Jay should not be able to view the quota of ATLAS. Only John and Mike
   can do that.
6. Duncan, the Manager of Visualisation should not be able to see the quota of
   CMS. Duncan should be able to see the quota of Visualisation only, and also
   the quota of any sub projects that will be created under Visualisation.
7. The total resources consumed by a project is divided into
     a) hard_limit - Maximum capacity that can be consumed.
     b) used - Resources used by the resource "volumes" in a project.
                      (excluding child-projects)
     c) reserved - Resources reserved for future use by the project does
                         not include descendants.
8. The quota information regarding number of volumes in different projects
   are as follows,

  +----------------+----------------+----------+--------------+
  | Name           | ``hard_limit`` | ``used`` | ``reserved`` |
  +================+================+==========+==============+
  |  ProductionIT  | 1000           |  100     | 100          |
  +----------------+----------------+----------+--------------+
  |  CMS           | 300            |  25      | 15           |
  +----------------+----------------+----------+--------------+
  |  Computing     | 100            |  50      | 50           |
  +----------------+----------------+----------+--------------+
  |  Visualisation | 150            |  25      | 25           |
  +----------------+----------------+----------+--------------+
  |  ATLAS         | 400            |  25      | 25           |
  +----------------+----------------+----------+--------------+
  |  Services      | 100            |  25      | 25           |
  +----------------+----------------+----------+--------------+
  |  Computing     | 200            |  50      | 50           |
  +----------------+----------------+----------+--------------+

  Consider following use-cases :-
  a. Suppose, Mike(admin of root project or cloud admin) increases the
     ``hard_limit`` of volumes in CMS to 400
  b. Suppose, Mike increases the ``hard_limit`` of volumes in CMS to 500
  c. Suppose, Mike delete the quota of CMS
  d. Suppose, Mike reduces the ``hard_limit`` of volumes in CMS to 350
  e. Suppose, Mike reduces the ``hard_limit``  of volumes in CMS to 200
  f. Suppose, Jay(Manager of CMS)increases the ``hard_limit`` of
     volumes in CMS to 400
  g. Suppose, Jay tries to view the quota of ATLAS
  h. Suppose, Duncan tries to reduce the ``hard_limit`` of volumes in CMS to
     400.
  i. Suppose, Mike tries to increase the ``hard_limit`` of volumes in
     ProductionIT to 2000.
  j. Suppose, Mike deletes the quota of Visualisation.
  k. Suppose, Mike deletes the project Visualisation.

9. Suppose the company doesn't want a nested structure and want to
   restructure in such a way that there are only four projects namely,
   Visualisation, Computing, Services and Operations.


Project Priority
-----------------

The code in the existing DBQuotaDriver is deprecated and hence we need an
update. Also as the entire OpenStack community is moving toward hierarchical
projects this can be an useful addition to Cinder.


Proposed change
===============

1. The default quota (hard limit) for any newly created sub-project is set to 0.
   The neutral value of zero ensures consistency of data in the case of race
   conditions when several projects are created by admins at the same time.
   Suppose the default value of number of volumes allowed per project is 100,
   and A is the root project. And an admin is creating B, a child project of A,
   and another admin is creating C, again a child project of A. Now, the sum
   of default values for number of volumes of B and C are crossing the default
   value of A. To avoid this type of situations, default quota is set as Zero.
2. A project is allowed to create a volume, only after setting the quota to a
   non-zero value (as default value is 0). After the creation of a new project,
   quota values must be set explicitly by a Cinder API call to a value which
   ensures availability of free quota, before resources can be claimed in the
   project.
3. A user with role "cloud-admin" in the root project and with inherited roles
   is called Cloud-Admin and is permitted to do quota operations across the
   entire hierarchy, including the top level project. Cloud-Admins are the only
   users who are allowed to set the quota of the root project in a tree.
4. A person with role "project-admin" in a project is permitted to do quota
   operations on its sub-projects and users in the hierarchy. If the
   role "project-admin" in a project is set as inheritable in Keystone, then
   the user with this role is permitted to do quota operations starting from
   its immediate child projects to the last level project/user under the
   project hierarchy.
   Note: The roles like "cloud-admin" and "project-admin" are not hard coded.
   It is used in this Blueprint, just for demonstration purpose.
5. The total resources consumed by a project is divided into

     a) Used Quota  - Resources used by the volumes in a project.
                      (excluding child-projects)
     b) Reserved Quota - Resources reserved for future use by the project does
                         not include descendants.
     c) Allocated Quota - Sum of the quota ``hard_limit`` values of immediate
                          child projects

6. The ``free`` quota available within a project is calculated as
         ``free quota = hard_limit - (used + reserved + allocated)``

   Free quota is not stored in the database; it is calculated for each
   project on the fly.
7. An increase in the quota value of a project is allowed only if its parent
   has sufficient free quota available. If there is free quota available with
   the parent, then the quota update operation will result in the update of
   the ``hard_limit`` value of the project and ``allocated`` value update of
   its parent project. That's why, it should be noted that updating the quota
   of a project requires the token to be scoped at the parent level.

   * Hierarchy of Projects is as A->B->C (A is the root project)

     +------+----------------+----------+--------------+---------------+
     | Name | ``hard_limit`` | ``used`` | ``reserved`` | ``allocated`` |
     +======+================+==========+==============+===============+
     |  A   | 100            |  0       |  0           |   50          |
     +------+----------------+----------+--------------+---------------+
     |  B   | 50             | 20       |  0           |   10          |
     +------+----------------+----------+--------------+---------------+
     |  C   | 10             | 10       |  0           |    0          |
     +------+----------------+----------+--------------+---------------+

     Free quota for projects would be:

     A:Free Quota = 100 {A:hard_limit} - ( 0 {A:used} + 0 {A:reserved} +
                         50 {A:Allocated to B}) = 50

     B:Free Quota = 50  {B:hard_limit} - ( 20 {B:used} + 0 {B:reserved} +
                         10 {B:Allocated to C}) = 20

     C:Free Quota = 10  {C:hard_limit} - ( 10 {C:used} + 0 {C:reserved} +
                         0 {C:Allocated}) = 0

     If Project C ``hard_limit`` is increased by 10, then this change results
     in:

     +------------+----------------+----------+--------------+---------------+
     | Name       | ``hard_limit`` | ``used`` | ``reserved`` | ``allocated`` |
     +============+================+==========+==============+===============+
     |  A         | 100            |  0       |  0           |   50          |
     +------------+----------------+----------+--------------+---------------+
     |  B         | 50             | 20       |  0           |   20          |
     +------------+----------------+----------+--------------+---------------+
     |  C         | 20             | 10       |  0           |    0          |
     +------------+----------------+----------+--------------+---------------+

     If Project C hard_limit needs to be increased further by 20, then this
     operation will be aborted, because the free quota available with its
     parent i.e. Project B is only 10. So, first project-admin of A should
     increase the ``hard_limit`` of Project B (using scoped token to
     Project A, because of action at level A) and then increase the
     ``hard_limit`` of Project C (again scoped token to Project B)

     Please consider the use cases mentioned above. The quota information
     of various projects, including the allocated quota is as follows,

     | ProductionIT  : hard_limit=1000, used=100, reserved=100, allocated=700
     | CMS           : hard_limit=300, used=25, reserved=15, allocated=250
     | Computing     : hard_limit=100, used=50, reserved=50, allocated=0
     | Visualisation : hard_limit=150, used=25, reserved=25, allocated=0
     | ATLAS         : hard_limit=400, used=25, reserved=25, allocated=300
     | Services      : hard_limit=100, used=25, reserved=25, allocated=0
     | Computing     : hard_limit=200, used=50, reserved=50, allocated=0

     * Suppose Mike tries to increase the volumes quota in CMS to 400.
       Since Mike is having the role of admin in ProductionIT which is the
       parent of CMS, he can increase the quota of CMS provided that the
       token is scoped to ProductionIT. This is required because the increase
       of quota limit in CMS results in the corresponding reduction of
       free quota in ProductionIT.

       Using the above formula, free quota of ProductionIT is given by,
       free quota = hard_limit - used - reserved - allocated
       free quota = 1000 - 100 - 100 - (400 + 400)
       free quota = 0

       So maximum permissible quota for CMS is 300 + 100 = 400

       Note:ProductionIT:allocated = CMS:hard_limit + ATLAS:hard_limit

       Minimum quota of CMS is given by,
       CMS:used + CMS:reserved + CMS:allocated = 25 + 15 + 250 = 290

       Note: CMS:allocated = Visualisation:hard_limit + Computing:hard_limit

       Since 290(minimum quota) <= 400(new quota) <=400(maximum quota),
       quota operation will be successful. After update, the quota of
       ProductionIT and CMS will be as follows,

       | ProductionIT : hard_limit=1000, used=100, reserved=100, allocated=800
       | CMS          : hard_limit=400, used=25, reserved=15, allocated=250

     * Suppose Mike tries to increase the intances quota in CMS to 500. Then
       it will not be successful, since the maximum quota available
       for CMS is 400.

     * Suppose Jay who is the Manager of CMS increases the volumes
       quota in CMS to 400, then it will not be successful, since Jay is not
       having admin or project-admin role in ProductionIT which is the parent
       of CMS.

     * Suppose Mike tries to increase the quota of ProductionIT to 2000,
       then it will be successful. Since ProductionIT is the root project,
       there is no limit for the maximum quota of ProductionIT. And also,
       Mike is having admin role in ProductionIT. For a private cloud the
       hard_limit depends on the internal inventory that is maintained
       internally by the cloud provider. Mike the Cloud Admin will have
       an access to these details and will update the hard_limit depending
       on the available inventory. So hard_limit is bounded by the available
       inventory for the Private Cloud and it will vary for each Private Cloud.

8. A decrease in the quota value of a project is allowed only if it has free
   quota available, free quota > 0 (zero), hence the maximum decrease in
   quota value is limited to free quota value.

 * Hierarchy of Projects is A->B->C, where A is the root project
      Project A (hard_limit = 100, used = 0, reserved = 0, allocated = 50)
      Project B (hard_limit = 50, used = 20, reserved = 0, allocated = 10)
      Project C (hard_limit = 10, used = 10, reserved = 0, allocated = 0)

      If Project B hard_limit is reduced by 10, then this change results in
      Project A (hard_limit = 100, used = 0, reserved = 0, allocated = 40)
      Project B (hard_limit = 40, used = 20, reserved = 0, allocated = 10)
      Project C (hard_limit = 10, used = 10, reserved = 0, allocated = 0)

      If Project B's hard_limit needs to be reduced further by 20, then this
      operation will be aborted, because the free quota of Project B should
      be greater than or equal to (20+0+10).

9. Suppose Mike tries to reduce the volumes quota in CMS to 350,
   it will be successful since the minimum quota required for CMS is 290.

10. Suppose Mike tries to reduce the volumes quota of CMS to 200,
    then it will not be successful, since it violates the minimum quota
    criteria.

11. Delete quota is equivalent to updating the quota with zero values. It
    will be successful if the allocated quota is zero. Authentication logic
    is same as that of update logic.

    * Suppose Mike tries to  delete the quota of CMS then it will not be
      successful, since allocated quota of CMS is non-zero.

    * Suppose Mike deletes the quota of Visualisation, then it will be
      successful since the allocated quota of Visualisation is zero. The
      deleted quota of Visualisation will add to the free_quota of CMS. The
      quota of CMS will be CMS :hard_limit=300, used=25, reserved=15,
      allocated=100.

    * Suppose, Mike deletes the project Visualisation, the quota of
      Visualisation should be released to its parent, CMS. But in the
      current setup, Cinder will not come to know, when a project is
      deleted from keystone. This is because, Keystone service is not
      synchronized with other services, including Cinder. So even if
      the project is deleted from keystone, the quota information
      remains there in cinder database. This problem is there in
      the current running model of OpenStack. Once the keystone service
      is synchronized, this will be automatically taken care of. For
      the time being, Mike has to delete the quota of Visualisation,
      before he is deleting that project. Synchronization of keystone
      with other OpenStack services is beyond the scope of this blueprint.

12. Suppose if Jay, who is the Manager of CMS tries to view the quota of
    ATLAS, it will not be successful, since Jay is not having any role in
    ATLAS or in the parent of ATLAS.

13. Suppose Duncan who is the Manager of Visualisation tries to update the
    quota of CMS, it will not be successful, because he is not having admin or
    project-admin role in the parent of CMS.

14. Suppose if the organization doesn't want a nested structure and wants
    only four projects namely, Visualisation, Computing, Services and
    Operations, then the setup will work like the current setup where there is
    only one level of projects. All the four projects will be treated as root
    projects.

15. In case of parallel quota consumption i.e. quota consumption by various
    sub-projects at the same time, if the request is not satisfied, then
    the request should be re-tried after sometime after increasing the parent
    quota. This needs admin intervention for this spec. As discussed in
    notifications impact section going ahead with notification/event based
    support we can overcome admin intervention. In upcoming work I
    will add details on how do we serialize request. This is
    important because increase in hard_limit for a subproject
    (depending on the free_quota in the parent) and updating the allocated
    quota at the parent level needs to be an atomic operation. Once the
    atomic operation is performed new incoming request can be served.

Alternatives
------------

For quota update and delete operations of a project, the token can be scoped to
the project itself, instead to its parent. But, we are avoiding that, because
the quota change in the child project lead to change in the free quota of the
parent. Because of that, according to this bp, for quota update and delete
operations, the token is scoped to the parent.


Data model impact
-----------------

Create a new column ``allocated`` in table ``quotas`` with default value 0. This
can be done by adding a migration script to do the same.


REST API impact
---------------

None


Security impact
---------------

The parameter hard_limit used in the spec needs to be closely tied
to the actual inventory present in each individual private cloud. As
most of the calculations are based of hard_limit the transparent view
of available inventory is needed for quota calculations. This is beyond
the scope of this spec but bringing it up for clarity. Also care
should be taken to ensure that this does not allow any quota escape
vulnerabilities. Given it is a more complex model, it is well worth
reviewers spending time on actively trying to subvert the
model (time of check v/s time of use, etc).


Notifications impact
--------------------

Any change of quota values (not usage, but limits) will
generate an event. This can be useful for general debugging,
auditing to figure out who did what. For this spec will restrict
the notification scope to just tracing and auditing but going ahead
event or notification mechanism can be used to notify the starving
sub-project, to retry later, when the parent has enough
free quota available. Also going ahead when Keystone deletes
a project a notification can go to a queue and then cinder
can consume this event to proactively free up the quota from
cinder db.


Other end user impact
---------------------

Only Cloud-Admin or immediate parent can set quota on sub-project.
Cloud-Admin can even set and update his/her own quota.


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
  * vilobhmm

Other contributors:


Work Items
----------

1. For the first implementation patch of this spec we will only
   fix up the quota calculations to be sub-project aware not add
   any new editing capabilities. Also we will skip updating of
   quotas by anybody except root project admin.

2. For second implementation patch of this spec we will add
   quota editing by project admin of sub-projects as well.

3. A role called "cloud-admin" will be created and assigning
   that role to a user in root project and making it inheritable.

4. Role called  "project-admin" will be created. The user with "project-admin"
   role in a project will be  able to do quota operations in projects
   starting  from its immediate child projects to the last level
   project/user under the project hierarchy, provided it is inheritable.

   Note:The roles like "cloud-admin" and "project-admin" are not hard coded.
   It is used in this Blueprint, just for demonstration purpose.

5. A new Quota Driver called ``NestedQuotaDriver`` will be implemented by
   extending the existing ``DbQuotaDriver``, to enforce quotas in hierarchical
   multitenancy in OpenStack.

6. A migration script will be added to create the new column ``allocated`` in
   table ``quotas``, with default value 0.


Dependencies
============

Depends on `bp hierarchical-multitenancy
<https://blueprints.launchpad.net/keystone/+spec/hierarchical-multitenancy>`_


Testing
=======

* Unit tests will be added for all the REST APIs calls.

* Add unit tests for integration with other services.


Documentation Impact
====================

Documentation will be updated to give details about the quota calculation and
how the quota will be assigned and managed by the projects while using
nested quota driver(in projects which have hierarchical support). These
deployment need to be on or later Kilo since the hierarchical support was
added since Kilo release.


References
==========

* `Hierarchical Projects Wiki <https://wiki.openstack.org/wiki/HierarchicalMultitenancy>`_

* `Hierarchical Projects
  <http://specs.openstack.org/openstack/keystone-specs/specs/juno/hierarchical_multitenancy.html>`_

* `Hierarchical Projects Improvements
  <https://blueprints.launchpad.net/keystone/+spec/hierarchical-multitenancy-improvements>`_

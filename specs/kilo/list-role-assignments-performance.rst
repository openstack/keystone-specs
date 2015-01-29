..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Improve List Role Assignments API Performance
=============================================

`bp list-role-assignments-performance
<https://blueprints.launchpad.net/keystone/+spec/list-role-assignments-performance>`_


As of the Havana release, keystone provides an API to list all existing role
assignments in the system. However, inheritance and group assignments expasion
occur at controller level in an inefficient way.


Problem Description
===================

Filtering on returned list of role assignments occurs at controller level,
after getting all available entities on the system from calling the manager and
driver levels, which is very inefficient. This is the first problem addressed
by this change proposal.

The second problem addressed is the expansion of inherited and group role
assignments, which occur at controller level, when it should be placed at
manager, since it is the level where additional business logic should be put.

Although not explicitly addressed in this spec, having a manager level method
that lists role assignments while taking account of filtering and inheritance,
would allow a number of other existing manager methods to use it rather than
duplicate the inheritance and group processing logic -- as it is today.

Proposed Change
===============

Basically, three main changes are proposed:

1. Propagate provided filters from controller to manager and driver levels;
2. Make the backend drivers consider filtering when executing list role
   assignment queries;
3. Move the code of expansion of inherited and group role assignments from the
   controller to the manager level.

In order to have a global view on how will be the flow of operations after the
proposed change, consider the following sequence of operations:

1. The controller validates and passes provided filters to the manager level;
2. The manager checks whether effective option is used and then makes
   performant requests to the driver;
3. The driver returns the filtered list of role assignments;
4. If in effective mode, the manager expands inherited and group role
   assignments and then returns the resultant list;
5. The controller formats the list of entities and returns the API response.


Alternatives
------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

The performance of the list role assignments API will be impacted positively.

In terms of memory use, since only the needed data will be coming from driver
to manager level, it will be significantly decreased in the cases where filters
by attributes are provided.

Regarding computational effort, since the manager will expand only inherited
and group role assignments that match the provided filters, it will be
significantly decreased as well.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

The list role assignments method at assignment manager will become the single
entry point for querying role assignments for a given user or group. It will be
specially used when issuing a token, which actually uses alternative methods to
compute effective role assignments.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Samuel de Medeiros Queiroz (samueldmq)

Other contributors:

* Henry Nash (henrynash)
* Rodrigo Duarte Sousa (rodrigods)

Work Items
----------

* In order to ensure the current behavior will not be changed by the
  proposed refactoring:

  * Implement and test restrictions on filters combinations;
  * Create a consistent suite of tests in order to assert the result from
    combining available filters.

* Regarding the refactoring implementation:

  * Pass the filters to the driver level and return only requested role
    assignments;
  * Migrate inherited and group role assignments expansion from controller to
    manager.


Dependencies
============

None


Documentation Impact
====================

None


References
==========

The patches that implement this specification are already under review and may
be found at the links below.

* `Check for invalid filtering on v3/role_assignments
  <https://review.openstack.org/#/c/144703/>`_

* `Improve List Role Assignment Tests
  <https://review.openstack.org/#/c/137021/>`_

* `Improve List Role Assignments Filters Performance
  <https://review.openstack.org/#/c/137202/>`_

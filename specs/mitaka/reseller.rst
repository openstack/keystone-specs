
..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================
Reseller Use Case
=================

`bp reseller <https://blueprints.launchpad.net/keystone/+spec/reseller>`_

OpenStack needs to grow support for hierarchical ownership of objects.
This enables the management of subsets of users and projects in a way that is
much more comfortable for private clouds, besides giving to public cloud
providers the option of reselling a piece of their cloud.

Problem Description
===================

Use Case 1:

* Resellers

**Actors**

* Alex - Cloud Owner

* Martha - Owner of ProductionIT

* Joe - Development Manager from WidgetMaster

* Sam - Development Manager from SuperDevShop

Alex is a owner of a cloud and Martha provides IT services to multiple
enterprise clients using resources that she bought from Alex. She would like to
offer cloud services to Joe at WidgetMaster, and Sam at SuperDevShop. Joe and
Sam have multiple QA and Development teams with many users. They need the
ability to create users, groups, projects, and quotas as well as the ability to
list and delete resources across their enterprises. Martha needs to be able to
set the quotas for both WidgetMaster and SuperDevShop. She also needs to ensure
that Joe and Sam cannot see or manipulate anything owned by each other.

Proposed Change
===============

Implement support to allow users/groups to be owned at more than just by a top
level domain. To achieve this, the domains construct will be merged with the
projects ones, since we already have a project's hierarchy, this merge will
allow the possibility to have also a domain hierarchy to distribute users and
groups not just in a global entity. With this implementation, we'll cover the
Reseller use case, where one will be able to resell both a project and a
project that behaves like a domain, where one can create users and groups.

We will take following steps in order to enable such feature:

* A new field will be added to the project table to represent if that project
  has the domain feature (``is_domain`` flag).

* An sql migration will take the following steps:

 ** Create, for each entry in the domain table, an entry in the project table
    where the domain's id becomes project_id in the project table with
    the ``is_domain`` flag set to true

 ** Initialize the parent_id as for any other projects that do not have
    parent_id set to match their domain_id

 ** Ceate a USER-PROJECT and a GROUP-PROJECT role assignment for each
    USER-DOMAIN/GROUP-DOMAIN existing

 ** Delete all domain assignments.

 ** Drop the domain table

* We are going to add the ``parent_id`` query parameter for both GET v3/domains
  and v3/projects APIs. If it is not specified, these requests should return
  only a list of root domains or projects.

It is also important to note the following rules/restrictions:

* A project with the "is_domain" flag set to "true" can only be either a root
  project or a child project to a project who also have the "is_domain" flag
  set to "true". Moreover, the "is_domain" flag is immutable.

* Regarding role assignments management, it will be possible to use the
  inherited role assignments to manage the grants between user/groups and the
  project hierarchy. For example, following the use case described in the
  previous session: by default, Martha or Alex can neither see nor manipulate
  the resources owned by Sam and Joe, unless she either explicitly has a role
  assignment in their domains or has an inherited role assignment in
  ProductionIT. In the same way Joe and Sam can not see anything owned by
  Martha unless she gives them a role on her domain.

* Note that today we can use the current role assignments mechanism to grant
  roles in the hierarchy for users and groups, for a better usability to manage
  the access control for the users and groups, we recommend the use of
  inherited roles assignments implementation, so you can grant a role to a
  user/group in a project/domain and inherit this assignment along the subtree.
  In addition, there is a `spec <https://review.openstack.org/#/c/133855>`_
  related to domain roles, that can improve this role management in
  Hierarchical Multitenancy. It is important to observe that the current
  implementation of inherited role assignments consider **only** projects, role
  assignments won't be inherited in a domain hierarchy.

* Creating a new non-root domain will be possible via both domains and
  projects APIs: POST v3/domains (create domain passing a parent_id from an
  existing domain) and POST v3/projects with the ``is_domain`` flag enabled and
  passing a parent_id from an existing domain.

* In this version, we will not allow to update a project to behave as a domain.

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

Python-keystoneclient must support creating projects that behave as a domain.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

* When a user requests a domain scoped token, Keystone will send a dual scoped
  token for domain and project.

Implementation
==============

Assignee(s)
-----------
Primary assignee:

Raildo Mascena <raildo>

Other contributors:

 Andrey Brito <abrito>

 Henrique Truta <henriquetruta>

 Rodrigo Duarte Sousa <rodrigodsousa>

Work Items
----------

1. "Domain is a project": Create a ``is_domain`` flag in the Project table to
   represent a domain;

2. Implement the ``parent_id`` query param for the list domains and list
   projects v3 API calls;

3. Migrate the current domain construct to the project table;

4. When a domain scoped token is requested for a project with the ``is_domain``
   flag active, a dual scoped token will be provided, referencing the project
   which holds that domain - we will have a single entry in the role
   assignments table with a USER_PROJECT or GROUP_PROJECT type.

* Note that we can have more than one domain or project with the same **name**
  so if one wants to request a token passing the entity name, would be
  necessary to pass the full namespace of that entity, for example considering
  the following hierarchy::

                                 A
                                / \
                               B   C
                              /     \
                             A       B

  To request a token for B, child of C (not the child of A), the token request
  would need to have the full hierarchy information.

* The character '/' won't be allowed to be part of a domain or project name, if
  an entity already has this character in its name, this specific entity won't
  be allowed to be part of a hierarchy. The name would need to be updated in
  order to remove the '/'.

5. Create a constraint to ensure that the parent of a domain will always be
   another domain (in other words: ensure that we won't a create a domain under
   a project);

6. Make the ``is_domain`` property immutable once it is enabled.

Dependencies
============

* Depends on Hierarchical Multitenancy improvements spec:
  https://review.openstack.org/#/c/135309/

Documentation Impact
====================

The Identity API v3 Documentation must be updated according to these changes.

References
==========

* `Kilo Summit Summary <https://www.morganfainberg.com/blog/2014/11/12/kilo-summit-summary/#hm>`_

* `Keystone Meetup Summit <https://etherpad.openstack.org/p/kilo-keystone-meetup>`_


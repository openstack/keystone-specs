..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Hierarchical Projects
=====================

`bp hierarchical-multitenancy
<https://blueprints.launchpad.net/keystone/+spec/hierarchical-multitenancy>`_

OpenStack will add support for hierarchical ownership of objects.

This enables the management of projects and quotas in a way that is more
comfortable for private clouds, because in a private cloud, you can organize
better your departmental divisions they work as "subprojects".

In short, the proposal is to modify the organizational structure of OpenStack,
creating nested projects in Keystone.

Problem Description
===================

Use Case 1:


* A division of a large enterprise is represented by a domain in an OpenStack
  installation,  and consists of Dev and Test teams.

* The division admin team wants to be able to assign quotas to each of the
  sub-teams for all their projects.

* The division admin team creates all the users for Dev and Test in the company
  LDAP, which the divisional domain references for authentication.

* The domain admin team creates a top level project for each of the Dev  and
  Test teams, and assign an admin from each team the ``project_admin`` role on
  their respective top level project. The domain admin team create a quota for
  each team on their respective top level project.
* Each team can then creates projects below their top level project, and the
  usage vs quotas can be compared at the top level project level.

Proposed Change
===============

1. Create the hierarchy:

* Example of the new project hierarchy::

   +-------------------------------------+
   |               Division A            |
   |                                     |
   |        +-------------------------+  |
   |        | User: domain_admin_team |  |
   |        | Role: domain_admin      |  |
   |        +-------------------------+  |
   |                                     |
   |                 /\                  |
   |                                     |
   |                /  \                 |
   |                                     |
   |               /    \                |
   |                                     |
   |              /      \               |
   |                                     |
   |             /        \              |
   |                                     |
   |            /          \             |
   |                                     |
   | +--------------+ +--------------+   |
   | |   Dev        | |     Test     |   |
   | +--------------+ +--------------+   |
   | +--------------+ +--------------+   |
   | | User: Joe    | | User: Sam    |   |
   | | Role: p_admin| | Role: p_admin|   |
   | +--------------+ +--------------+   |
   |                                     |
   |       /                  \          |
   |                                     |
   |     /                      \        |
   |                                     |
   | +---------------+ +----------------+|
   | | Dev.subproject| | Test.subproject||
   | +---------------+ +----------------+|
   |                                     |
   | p_admin= role project_admin         |
   +-------------------------------------+

  * After that you must create domains and the projects hierarchies will be
    placed under those domains. You can create as many domains as you want and
    as many hierarchies as you want under each domain.

2. Max Depth Tree:

   * As of the first release we should have a configuration option allowing to
     restrict the depth of the tree with a reasonable default of 5.

3. Update Projects:

   * In this first release, It will not be possible to update the hierarchy.
     So we can't change the parent project of any project.

4. Delete Projects:

   * It is possible to delete leaf projects.

   * The first version will support a non-recursive delete function which will
     fail with "in use" or similar if the project to be deleted has children.

5. Get Projects:

   *  Clear identifier to indicate we are looking for hierarchy details.

6. Roles:

   * Inherited roles assignments:
     If a user has, say, a role assignment "project_member" that was marked as
     inherited in a project, then this user will automatically have this role
     on any child projects. Currently, inherited roles assignments only work
     from domains to projects, this proposal expands this inheritance to work
     down a hierarchy of projects.

   * This change will be implemented in the extension OS-INHERIT, like
     currently working for domains.

   * Example:

     * The domain_admin_team creates the Dev and Test projects and assigns the
       role ``project_admin`` to project_admin_user. As their role is
       inheritable it will have access to their children.

     * As Joe has ``project_admin`` role assignment in Dev project, he can
       create instances in this project and can create subproject and control
       quotas to his subprojects. The same thing will happen to Sam in Test.

     * The user_project_admin can grant/revoke roles to users in its project
       and in its subprojects. A user with a member role can't grant/revoke
       roles.

7. Token:

   * Token must be scoped to the target project on which the action is
     performed.

   * If the role assignment of a project is inheritable, tokens granted to
     child projects will also contain this role assignment, otherwise it will
     not have access.

8. Users:

   * This proposal does not change user/group management - this is still
     handled at the domain level.

Notes:

* Not available in Keystone V2 API.

Alternatives
------------

None

Data Model Impact
-----------------

Create a new column “parent_project_id” in table “project”, when the column
is null, it means that this project is the root of the tree.

REST API Impact
---------------

* The changes in API are defined in a separate review.
  https://review.openstack.org/#/c/111355/

Security Impact
---------------

A user will only have access to projects which he has a directly assigned
role or a role inherited from a parent project.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * raildo

Other contributors:
  * schwicke
  * sajeesh
  * tellesnobrega
  * rodrigodsousa
  * afaranha
  * henrique-4
  * thiagop
  * gabriel-bezerra
  * samuel-z


Work Items
----------

1. Projects in Keystone will gain a new field, parent_project_id, to create a
   tree of projects. (This change will be made in Keystone core)

2. Role assignments will be inherited down the project hierarchy tree but
   currently, inherited role assignments are only supported from domain to
   project. So we have to create a API for inherited roles for projects, we
   have to implement the following functions: (This change will be made in
   extension OS-INHERIT)

  * A role assignment defined for a project A must be inherited by all the
    child projects of project A;
  * A role assignment defined for a group X must be inherited by all the child
    projects of that group X;
  * For a project A, list all the inherited roles assignment of A, which will
    also be inherited by the child projects of project A;
  * For a group X, list all the inherited role assignment of ABC projecs, which
    will also be inherited by the users in the group X in the child projects;
  * Check if a user has an inherited role assignment on a project;
  * Check if a group has an inherited role assignment on a project;
  * Revoke an inherited project role assignment from a user on a project;
  * Revoke an inherited project role assignment from group on a project.

3. Currently GET /role_assignments is extended by OS-INHERIT to return role
   assignments that are inherited from domain to project. This proposal will
   further extend this to also include role assignments inherited from project
   to project. (This change will be made in Keystone core and the OS-INHERIT
   extension)

4. Update the token contents to include roles inherited down the project
   hierarchy. (This change will be made in Keystone core)

5. We will create a call in the API and python-keystoneclient to return the
   hierarchy with the options: (This change will be made in Keystone core)

  * Parent projects
  * Children projects
  * Full hierarchy

Dependencies
============

None


Testing
=======

* Add unit tests for integration with other services. (Tempest Tests)


Documentation Impact
====================

The new ways to manipulate hierarchical projects must be documented in the API.

References
==========
* `Wiki <https://wiki.openstack.org/wiki/HierarchicalMultitenancy>`_

* `Juno Etherpad <https://etherpad.openstack.org/p/juno-keystone-hierarchical-multitenancy>`_

* `Inherited Roles <http://docs.openstack.org/api/openstack-identity-service/3/content/api-1.html>`_

* `API Reference <http://docs.openstack.org/api/openstack-identity-service/3/content/api-1.html>`_

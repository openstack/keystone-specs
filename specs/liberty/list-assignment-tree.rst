..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
List Assignments for a Project Tree
===================================

`bp list-assignment-subtree <https://blueprints.launchpad.net/keystone/+spec/list-assignment-subtree>`_


Provide the ability to list assignments for all projects in a given tree.


Problem Description
===================

We already support the capability of a hierarchy of projects, along with some
APIs that can operate on such a tree (for example, listing all the projects in
that tree as a list). Such APIs are provided to simplify the work of other
projects using keystone - for example nova managing quotas for a tree.

The Horizon team (and hence likely other UI developers) are also starting to
integrate project-tree operations into their support. One immediate request
that came out of this work was the ability to list all role assignments in a
domain.

Proposed Change
===============

While it would be possible to provide a domain-specific ability to list
assignments for those projects within the domain, it is worth first stepping
back and considering this requirement in the light of the changes we are
making to domains and project hierarchies. The proposals being considered
for Liberty (which have been discussed at length at the last two summits) is
that a domain becomes a special type of project, and all projects within that
domain are children within that project's hierarchy. Given this trajectory, it
would seem more appropriate that we provide the ability to list assignments
for a project hierarchy, and if that project is acting as a domain then this
will result in listing all assignments for the domain.

The advantage of this approach is that it keeps with our goal of moving towards
a project-related API for domain operations (although we are, of course,
maintaining support for the legacy domain API), as well as provides the ability
for listing assignments for any arbitrary project tree, giving more options to
API clients in how they present keystone objects.

Like other tree operations we support, for now a domain boundary will be
opaque - so calling the new API on anything other than a leaf domain will not
include any assignments related to sub-projects.

Alternatives
------------

We could provide something specific to domains, although other than removing
the dependency on the domain-project integration work already in flight for
Liberty, there seems little other advantage.

Data Model Impact
-----------------

None

REST API Impact
---------------

To provide the API, an additional query parameter (``include_subtree``) will
be added to the ``GET /role_assignments`` call.

Security Impact
---------------

In terms of policy for the new API, it is proposed that this is a separate
policy rule than for the regular list assignments rule. This would allow the
API to be permissioned by a role on the root project of the tree upon which the
API was called. This is compatible with the most common use case of this new
API of a domain admin examining the assignments for all projects in a domain.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

The performance of listing assignments in a tree of projects will likely be
worse than listing within a single project. However, the implementation will
be able to take advantage of other work to move the filtering of listing
assignments into the driver as well as implementing a materialized tree::

`Improve List Role Assignments Filters Performance <https://review.openstack.org/#/c/137202>`_
`bp materialize-project-hierarchy <https://review.openstack.org/#/c/173424/>`_


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
    henry-nash

Work Items
----------

- Implement the new API options
- Add support to keystoneclient library
- Add support to openstack client

The work for supporting this API in Horizon will be proposed separately.

Dependencies
============

None

Testing
=======

None, beyond the regular unit testing.

Documentation Impact
====================

Changes to the Identity API.

References
==========

None

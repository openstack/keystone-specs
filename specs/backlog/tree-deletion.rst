..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Project Tree Deletion
=====================

`bp project-tree-deletion
<https://blueprints.launchpad.net/keystone/+spec/project-tree-deletion>`_


Problem Description
===================

In the first version of Hierarchical Multitenancy (kilo-1), a project deletion
was allowed only for leaf projects in the hierarchy. With this restriction, it
is necessary to delete projects one by one (starting from the leafs) in order
to delete a whole hierarchy branch.

For example, considering the following hierarchy::

    +------------------------+
    |           A            |
    |                        |
    |        /      \        |
    |                        |
    |       /        \       |
    |                        |
    |      B          C      |
    |                        |
    |    /   \       /  \    |
    |                        |
    |   /     \     /    \   |
    |                        |
    |  D       E   F      G  |
    +------------------------+

To delete project B and its subtree, the user would need to first delete
projects D and E.

In addition, the subtree deletion/disabling needs to be performed as an atomic
operation.

Proposed Change
===============

This spec proposes the implementation of an API to perform a whole tree
deletion, which will allow the user to specify a project that is not a leaf in
the hierarchy for removal. In this way, the project itself along with its whole
subtree will be deleted.

In order to delete a project, it must be first disabled. Currently, this
follows the same behavior as the project deletion: the user would have to
manually disable the project's children to disable a project that is in a
higher level of the hierarchy (the project hierarchy doesn't follow the same
behavior as deleting domains, which has cascade effect by default). Thus, this
spec also proposes changes to allow the disabling of a whole hierarchy branch
within a single operation:

* Cascade enabling/disabling of projects: once a project in a higher level
  of the hierarchy is disabled/enabled, all projects in its subtree are
  disabled/enabled. This approach follows the current rule applied in the
  project hierarchy, where we can't have a disabled project with enabled
  children. This approach has the downside of triggering multiple writes in
  the database.

Both features lead to access privileges concerns: a user could have delete and
update privileges for a project in a higher level of the hierarchy and not to
some projects in its subtree. Although we could insist that the caller of a
recursive operation has the delete_project and update_project permissions on
every project, it is more likely that in practice a cloud provider will want to
reserve such operations to a very restricted set up users/roles. For this
reason, there is the need for new rules in the keystone ``policy.json`` file
and, therefore, new API endpoints to represent these actions.

The new APIs for the delete and update requests can have the following format:

* Delete: DELETE projects/<project_id>/cascade
* Update: PATCH projects/<project_id>/cascade

Specifying ``cascade`` indicates that the request should also be applied to all
children in the hierarchy.  Note that for the update request, the ``enabled``
field is mandatory and the **only** accepted attribute. Including other
attributes will raise an error - this results in the need of creating
additional calls in keystone's controller layer to be protected by the new
rules.

With the `Reseller <https://review.openstack.org/#/c/139824/>`_ spec, domains
are projects, in this way we have the following behavior:

* We will be able to have a hierarchy of projects that behaves as domains.
  Since a project acting as domain can only appear as the root of a hierarchy
  of regular projects (and never in the middle), the update and delete options
  on the hierarchy will now operate like their domain API equivalent;
* By default we will only allow the recursive behavior to affect "pure"
  projects (``is_domain = false``). If we trigger the recursive actions on a
  "pure" project we are certain it will never hit a domain as per consequence
  of the point above. This behavior can be changed by updating the rule in
  keystone's policy file.

For **leaf** domains, the ``cascade`` APIs have the same effect as the regular
update and delete domain operations, but they will be enforced by **different**
rules in the policy file.

Alternatives
------------

Use a different approach for enabling/disabling:

* Consider a whole branch as being **effectively** disabled once a project in
  a higher level of the hierarchy is disabled. By using this approach we avoid
  having to perform multiple writes in the database but we rely that revocation
  events are going to carry the information about the whole branch for token
  validation. In addition, when providing a token we need to check if the
  target has any disabled parents in the hierarchy.

Don't add a new rule in the policy file:

* Have a configuration where the delete and update behavior could be chosen
  (recursive or not). This leads to access privileges problems, an alternative
  could be to check if the user performing the request has access to all
  projects in that hierarchy branch prior to triggering the action.

Security Impact
---------------

New rules in the policy.json file to control the access to the recursive
deletion and disabling of projects/domains.

Notifications Impact
--------------------

A pycadf notification should be triggered for each project that is deleted or
**effectively** disabled/enabled - they will be triggered from child to parent,
this way the quota driver in other service can process the notification
properly when freeing the quota to its parent project.

Other End User Impact
---------------------

New behavior when deleting/disabling a project/domain along with new rules in
the policy.json file.

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
  * Rodrigo Duarte rodrigodsousa

Other contributors:
  * Raildo Mascena raildo
  * Henrique Truta henriquetruta

Work Items
----------

* Update API spec documentation;

* Add new rules to policy.json file;

* Add new endpoints to mirror the new features;

* Implement the new deletion/disabling behavior for the project's hierarchy.

Dependencies
============

None

Documentation Impact
====================

API Documentation (Identity API v3)

References
==========

* `HM Kilo Summit <https://etherpad.openstack.org/p/hierarchical-multitenancy-kilo-summit>`_
* `Reseller`
* `PyCADF <http://docs.openstack.org/developer/pycadf>`_


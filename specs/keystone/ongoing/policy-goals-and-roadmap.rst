..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Policy Goals and Roadmap
========================

Policy in OpenStack has long been the forefront of operator concerns and pain.
The implementation is complicated to understand, inconsistent across projects,
and lacks secure defaults.

The purpose of this document is to define the overall goals we need to achieve
with OpenStack's policy implementation and level-set on policy terminology, as
well as to describe the steps necessary to achieve those goals and to to close
security flaws within the current Role Based Access Control (RBAC)
implementation used by OpenStack.

Problem Description
===================

Identity and access management systems are responsible for allowing and denying
access to resources of the service based on identities. One way to achieve this
is by using Role Based Access Control (RBAC). RBAC is a security model that
allows access to resources through roles, which can be assigned to users or
identities.

OpenStack services base many of their operations within the context of a
project. For our purposes, projects can be thought of as logical containers of
resources. For example, an instance is owned by a project, meaning the instance
should be accessible from within that project but not outside of it. This helps
OpenStack model multiple organizations within a single deployment while
maintaining a level of separation or multi-tenancy. A side-effect is that this
introduces an additional level of complexity to OpenStack's access control
model. Not only does the entire system need to be aware of identities, roles,
and assignment, but it also needs to properly account for scope of specific
operations.

As a result, OpenStack relies on a derivative of the RBAC model unofficially
referenced as scoped-RBAC.

The scoped-RBAC model builds upon all assumptions of traditional RBAC, but adds
the concept of scope to the implementation. This would be the ideal model for
OpenStack if all operations within OpenStack aligned nicely within projects.
However, this is not the case. To highlight this more effectively, let's use
some examples.

Example One
-----------

Let's assume we would like to create an instance. We've already established
that instances should be contained within projects, so it makes sense to apply
the scope of the project we want to own the instance in to the request for
creating the instance (e.g. Bob wants to create an instance in the `Staging`
project, so first he gets a token scoped to the `Staging` project to send with
his compute API request for an instance).

This example clearly shows how scoped-RBAC works well with resource operations
that map properly to project scope.

Example Two
-----------

Let's look at an example of the opposite case. Let's assume we would
like to live migrate an instance from one compute host to another so that we
can perform some maintenance. That host could be responsible for instances that
are owned by different projects. Which project should you scope to such that
you can live migrate instances off a host for maintenance without violating
multi-tenant separation of concerns?

This is an example of an operation that doesn't fit well within the construct
of a project. Instead, live migration is more of a system-wide operation that
is independent of a specific project.

Summary
-------

Some operations within OpenStack work nicely with the concept of project scope,
while others do not. Further reading and context on this can be found in
`bug 968696 <https://bugs.launchpad.net/keystone/+bug/968696>`_, which
highlights the issues of scoped-RBAC for system-wide OpenStack operations and
resources.

One of the more problematic side-effects of the existing implementation that
operators have to deal with is choosing the lesser of two evils when violating
the principal of least privilege. The principal of least privilege states that
a user should have only the permissions necessary to carry out their duties and
nothing more. The current gap between system-scoped and project-scoped
operations or resources forces operators to either grant too many permissions
for the user or not give the user enough permissions to effectively do their
job. It's apparent that granting too many permissions violates the principal of
least privilege, but it's arguably true for the inverse, since too much
restriction doesn't let the user effectively do their job.

Goals
=====

The following goals increase OpenStack adoption by reducing complexity,
allowing better interoperability, and closing security gaps.

1. Administrative rights, via the `admin` role, should be treated consistently
   across OpenStack. For example, granting an administrator role to a user on a
   project shouldn't result in the ability to modify endpoints because
   endpoints aren't a project-specific resource. Likewise, granting a user
   administrative rights on the deployment system shouldn't result in the
   ability to do administrative things in a specific project.
2. The mapping of policies to operations should be easy to understand,
   customize, and maintain.
3. A variety of sensible roles should be available by default upon
   installation, or upgrade. These roles should correspond to the default
   policies provided by each project. This will require cross-project
   communication to ensure the roles make sense across project boundaries.

The list above allows operators to more easily accomplish use-cases such as:

* Grant users read-only access to projects
* Grant users administrator permissions on a per-project basis
* Grant users access to a subset of resource types for a particular project

Future Goals
------------

The following goals have been brought up in discussions several times. At this
point we are not ruling out the possibility of addressing these goals, but
formally recognizing them as something to be addressed after completing the
goals listed in the previous section (e.g. you must crawl before you can walk,
and walk before you can run).

1. It should be possible to delegate fine-grained access of a resource to
   another user or system.
2. It should be possible to build custom policies that can take into account
   users, resources, and scopes. This could result in policy being fed
   different assertions on specific resources. For example, having a policy
   that says a specific user can access my instance only on specific days and
   only if they have specific assertions available in the SAML document they
   used to authenticate.
3. It should be possible to determine what operations are available in a
   deployment while taking role assignments into consideration. This needs to
   be done in such a way that doesn't duplicate information across OpenStack
   services or increase maintenance overhead for operators.

Cross-Project Impact
====================

Policy and Role Based Access Control (RBAC) has traditionally been thought of
as a problem space within the OpenStack Identity umbrella. While that statement
logically makes sense, the current implementation of policy is enforced across
the OpenStack services making the ownership of the problem harder to pinpoint.
Keystone currently only acts as an information point to the service, which is
ultimately responsible for enforcing the policy decision based on information
from keystone and information from the project. The OpenStack community has
made several attempts to address these issues (linked below in the References
section), but having the implementation spread across OpenStack makes it
impossible for a single project to propose and fix these issues without
cross-project buy-in.

Cross-Project Resolution
========================

If a proposed solution to one of the above goals requires a coordinated effort
between projects, we will use either `community goals <https://governance.openstack.org/tc/goals/>`_,
`tags <https://governance.openstack.org/tc/reference/tags/index.html>`_,
or both. These tools require cross-project communication, buy-in, and
commitment.

For example, one community goal might be to define a set of default roles and
another to implement them consistently across services. Once a project tests
and implements the standardized default set, they can `assert:basic-rbac` as a
project tag.

These tools weren't available when previous solutions were proposed. Now that
we can use them as a community, they are a natural fit for solving complex,
distributed problems like consistent RBAC enforcement.

Road Map
========

This road map attempts to describe the steps necessary to mitigate the issues
outlined above by painting a picture of what the long term vision of policy in
OpenStack should be. Each of the following sections denotes a piece of work in
a sequence that results in a desired end state.

Reclassifying Operation & Scope
-------------------------------

Based on the example operations in the previous section, a good starting place
might be reassessing each operation and what the ideal scope should be. The
majority of this work could be considered preparation for consuming proper
scope from an identity system. Ideally, the responsibility of the service in
this case should be to compare the operation being done on a resource to a
scope provided by the identity system (e.g. does the instance being modified in
this request belong to the project the token is scoped to).

Work Items
~~~~~~~~~~

The following work items are needed to satisfy this item:

* Split the role check from the scope check in each service.
* Isolate the logic for scope checks into consistent patterns or in a
  consumable library.
* Assess each operation and associate which scope, or scopes, are appropriate.

Communicating Difference in Scope
---------------------------------

The next step, which could be worked in parallel to the previous step, is
communicating proper scope. In the current system, only two scopes are really
represented within OpenStack. The first is `unscoped`, which can be thought of
as a valid authentication with an empty set of operations. The second is
`project-scoped`, which is a valid authentication of a user with a role on a
specific project. These scopes are the most commonly used scopes within
OpenStack. However, the identity system supports two additional scopes. One is
called `domain-scoped`, which is a valid authentication of a user with a role
on a domain. The other is called `system-scope`, which is a valid
authentication of a user with a role on the deployment system and meant to be
used to access system specific resources.

Consuming Libraries
-------------------

Now that we have a way to assign roles on the system and scope tokens to that
context, we need to ensure the consuming libraries handle these changes
properly. Most OpenStack projects consuming tokens from keystone to enforce
policies do so through the use of `oslo.context`. The token response is used to
build a context dictionary using `oslo.context`. The project uses attributes of
the context dictionary to make assertions about authorization scope. We need to
ensure `oslo.context` properly handles this new scope.

Once those changes are made, the keystone team should be in a great place to
help other projects consume these changes. This might consist of helping other
projects separate their scope check from the role check or classifying
different scopes required for operations.

Work Items
~~~~~~~~~~

* Add support to `oslo.context` and `oslo.policy` for consuming system-scoped
  tokens and relaying that information in generated contexts for projects to
  consume.

Conclusion
==========

The end results should consist of an easy-to-use mechanism for granting system
roles, a clear interface for denoting scope to services, and a straight forward
policy pattern that projects can use to evaluate scope.

References
==========

The following are references to past or present specifications:

* `RBAC <http://csrc.nist.gov/groups/SNS/rbac/>`_
* `Admin-ness bug <https://bugs.launchpad.net/keystone/+bug/968696>`_
* `Role Check from Middleware specification <https://review.openstack.org/#/c/391624>`_
* `Tokens with subsets of roles specification <https://review.openstack.org/#/c/186979>`_
* `Role Check on Body Key specification <https://review.openstack.org/#/c/456974>`_
* `Dynamic RBAC Policy specification <https://review.openstack.org/#/c/279379>`_
* `Policy Merge specification <https://review.openstack.org/#/c/295049>`_
* `Fetch Policy by Tag specification <https://review.openstack.org/#/c/298788>`_
* `Policy rules managed from a database specification <https://review.openstack.org/#/c/133814>`_
* `Add policy-remove-scope-checks specification <https://review.openstack.org/#/c/433037/>`_
* `Add additional-default-policy-roles specification <https://review.openstack.org/#/c/427872/>`_
* `Add policy-docs specification <https://review.openstack.org/#/c/433010/>`_
* `Capability API spec <https://review.openstack.org/#/c/386555/>`_
* `System Scope Specification <https://review.openstack.org/#/c/464763/>`_

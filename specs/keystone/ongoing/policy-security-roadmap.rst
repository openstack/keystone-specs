..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Policy Security Roadmap
=======================

Policy in OpenStack has long been the forefront of operator concerns and pain.
The implementation is complicated to understand, inconsistent across projects,
and exposes security issues across OpenStack.

The purpose of this document is to describe the steps necessary to close
security flaws within the current Role Based Access Control (RBAC) of
OpenStack. Roadmaps to improve delegation usecases and policy will be covered
in a separate document.

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
specific project. A third scope exists, but is rarely used outside of the
identity service and that is `domain-scoped`. There isn't an existing mechanism
to denote `system-scope`, nor does the assignment mechanisms within identity
allow for it.

The first step of this item would be to advertise proper `system-scope`. This
requires keystone to understand system scope. One way to do this would be to
allow the use of system role assignments. Currently, role assignments must
consist of a user or group on a project or a domain. This would have to change
to allow for assignments on a `system` scope. In addition to allowing roles to
be assigned on the system, the existing scope mechanisms will need to be
modified to allow users to obtain system-scoped tokens.

The following are assumptions about the system scoping mechanism:

* A request for a system-scoped token returns a token with
  `token.system={'all'=True}` for a specific role or roles.
* System scoping should not be piggy-backed onto unscoped requests. Doing so
  could lead to security vulnerabilities given the existing usage pattern of
  unscoped tokens today.

Work Items
~~~~~~~~~~

* Make it so assignments can be set on the system (e.g. `openstack role add
  --user Bob --system observer`) in both the client and identity server.
* Introduce a new scoping mechanism that distinguishes `system-scope` from
  `project-scoped`, `domain-scoped`, and `unscoped` requests.

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
* `System Scope Specification <https://review.openstack.org/#/c/464763/>`_

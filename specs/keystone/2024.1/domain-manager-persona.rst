..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Domain Manager Persona for domain-scoped self-service administration
====================================================================

`bug #2045974 <https://bugs.launchpad.net/keystone/+bug/2045974>`_

In scenarios where customers are assigned a whole domain, they might desire
self-service functionality for managing users, projects and groups within that
domain.
The `Consistent and Secure Default RBAC
<https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html>`_
introduced the ``manager`` role for projects as a customer-side management role
between ``admin`` and ``member`` but added no such role model for domains.
This specification introduces a new ``domain-manager`` persona to fill that gap
and enables domain-scoped identity self-service capabilities for customers.


Problem Description
===================

Today, you need the ``admin`` role in keystone in order to fully manage groups,
projects and users including their role assignments. Since this role is
intended for the highest level of authorization, it is usually reserved for
operators. This means that even if a customer is confined within a domain in
Keystone, they need to involve operators of the cloud to have users, projects
and groups as well as corresponding role assignments managed for them.

For customers receiving a whole domain in Keystone this is often unsatisfactory
since a self-service capability is desired. An example use case for this is the
"virtual private cloud" (VPC) approach where the operator of a public cloud
wants to allocate reseller domains to customers which they can then subdivide
on a self-service basis.

For this purpose a role or persona and corresponding set of permissions are
required which implement this management functionality for customers.


Proposed Change
===============

Adjust existing policy defaults to incorporate the domain-scoped ``manager``
role accordingly like described below.

The ``domain-manager`` persona should be implemented in Keystone for managing
identity objects such as users, projects and groups along with role assignments
within a domain to enable identity management self-service for end users.

All default policies that concern users, projects, groups and roles in the
Identity API have to be adjusted to incorporate the ``domain-manager`` persona
on a domain level.
Below is an example of the ``identity:create_project```policy adjusted for the
``domain-manager`` persona. It adjusts check to accept the domain-scoped
``manager`` role in addition to ``admin``:

.. code-block:: python

   # this extends the current default (base.RULE_ADMIN_REQUIRED) by
   # the domain-scoped manager
   ADMIN_OR_DOMAIN_MANAGER = (
       '(' + base.RULE_ADMIN_REQUIRED + ') or '
       '(role:manager and domain_id:%(target.project.domain_id)s)'
   )

   # policy defintions concerning the management of users, projects, groups
   # and role assignments have to be adjusted to use this new rule,
   # for example the "identity:create_project" rule:
   policy.DocumentedRuleDefault(
       name=base.IDENTITY % 'create_project',
       check_str=ADMIN_OR_DOMAIN_MANAGER,
       scope_types=['system', 'domain', 'project'],
       description='Create project.',
       operations=[{'path': '/v3/projects',
                    'method': 'POST'}],
       deprecated_rule=deprecated_create_project),

For actions that address multiple target resources, additional checks need to
be added to ensure that only resources of the same domain are accessible to
the domain manager.
The following example illustrates this for the ``add_user_to_group`` policy
definition in the policy file for groups:

.. code-block:: python

   # copied over from grant.py
   DOMAIN_MATCHES_USER_DOMAIN = 'domain_id:%(target.user.domain_id)s'
   DOMAIN_MATCHES_GROUP_DOMAIN = 'domain_id:%(target.group.domain_id)s'

   # new rule for domain managers for user and group resources within domain
   DOMAIN_MANAGER_AND_DOMAIN_SCOPED_USER_GROUP_TARGETS = (
       '(role:manager and ' + DOMAIN_MATCHES_USER_DOMAIN + ' and'
       ' ' + DOMAIN_MATCHES_GROUP_DOMAIN + ')'
   )

   # the following policy definition has been modified to extend check_str
   # by the new rule introduced above
   policy.DocumentedRuleDefault(
      name=base.IDENTITY % 'add_user_to_group',
      check_str='(' + base.RULE_ADMIN_REQUIRED + ') or ' +
                DOMAIN_MANAGER_AND_DOMAIN_SCOPED_USER_GROUP_TARGETS,
      scope_types=['system', 'domain', 'project'],
      description='Add user to group.',
      operations=[{'path': '/v3/groups/{group_id}/users/{user_id}',
                   'method': 'PUT'}],
      deprecated_rule=deprecated_add_user_to_group)

One crucial part of a domain manager's responsibility will be to assign roles
within a domain. For this purpose, the existing rule defaults in Keystone can
be repurposed:

.. code-block:: python

   # below "GRANTS_DOMAIN_ADMIN" has been renamed to "GRANTS_DOMAIN_MANAGER"
   # and "role:admin" has been replaced by "role:manager"
   GRANTS_DOMAIN_MANAGER = (
       '(role:manager and ' + DOMAIN_MATCHES_USER_DOMAIN + ' and'
       ' ' + DOMAIN_MATCHES_PROJECT_DOMAIN + ') or '
       '(role:manager and ' + DOMAIN_MATCHES_USER_DOMAIN + ' and'
       ' ' + DOMAIN_MATCHES_TARGET_DOMAIN + ') or '
       '(role:manager and ' + DOMAIN_MATCHES_GROUP_DOMAIN + ' and'
       ' ' + DOMAIN_MATCHES_PROJECT_DOMAIN + ') or '
       '(role:manager and ' + DOMAIN_MATCHES_GROUP_DOMAIN + ' and'
       ' ' + DOMAIN_MATCHES_TARGET_DOMAIN + ')'
   )

   # the following has been renamed from "ADMIN_OR_DOMAIN_ADMIN" to
   # "ADMIN_OR_DOMAIN_MANAGER" to reflect the changed behavior
   ADMIN_OR_DOMAIN_MANAGER = (
       '(' + base.SYSTEM_ADMIN + ') or '
       '(' + GRANTS_DOMAIN_MANAGER + ') and '
       '(' + DOMAIN_MATCHES_ROLE + ')'
   )

However, a domain manager should not be able to assign roles of higher
privileges than themselves, so the set of target roles they are able to
assign/revoke should be restricted using a new rule. Defining this as a rule
offers the advantage that operators or deployers may easily adjust this role
list (which roles are manageable) without the need to rewrite the whole set of
lenghty individual rules for all the API actions:

.. code-block:: python

   # define a new rule called "domain_managed_target_role"
   policy.RuleDefault(
       name='domain_managed_target_role',
       check_str="'manager':%(target.role.name)s or "
                 "'member':%(target.role.name)s or "
                 "'reader':%(target.role.name)s"),

Finally, all the rules that concern role grants should be adjusted to
incorporate this target role restriction. Below is an example for
``identity:create_grant``:

.. code-block:: python

   # here a "and rule:domain_managed_target_role" is added to the check
   policy.DocumentedRuleDefault(
       name=base.IDENTITY % 'create_grant',
       check_str=(ADMIN_OR_DOMAIN_MANAGER +
                  ' and rule:domain_managed_target_role'),
       scope_types=['system', 'domain'],

The changes illustrated above need to be applied to all applicable policy
definitions that handle relationships between users, projects, groups and roles
within domains accordingly.

This spec does not address services other than Keystone. Introducing the
persona definition and role in Keystone lays the foundation for the
``domain-manager`` persona to be established in other services, such as Nova or
Cinder. However, the intended purpose and resulting permission set of a
``domain-manager`` persona is highly service-specific and up to the
corresponding projects to define and implement.
For Keystone this concerns domain-level identity management only, which is the
scope of this spec. Other services may adopt the ``domain-manager`` persona and
role in the future.

Alternatives
------------

A new role: ``domain-manager`` could also be used for this purpose.
The positive aspect would be a better differenciation between the ``manager``
on a project level and a ``manager`` on a domain level. For an end user it
might be more obvious which of the two use cases they have been assigned the
role for without looking up the assignment scope by themselves.
But a new ``domain-manager`` role would not fit into the new RBAC system, that
requires the roles to be hierarchically structured, when they can be assigned
to a user.

Another alternative would be to assign the ``admin`` role to end users within
domains in a scoped fashion like the current rule defaults imply. However, the
``admin`` role does not seem be properly scoped across all OpenStack services
(see `this Launchpad bug <https://bugs.launchpad.net/keystone/+bug/968696>`_).
Furthermore, `there has been operator
feedback <https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#the-issues-we-are-facing-with-scope-concept>`_
that a scoped ``admin`` role is a confusing concept in general. It seems to be
more appropriate to introduce a dedicated role for this, akin to the
``manager`` role for projects.

Security Impact
---------------

The ``domain-manager`` persona will allow customers to have administrative
management capabilities for users, projects, groups and role assignments within
a domain. However, the persona must first be assigned to a customer
user by an operator. This is a deliberate action by the operator and customers
do not get the persona or its permission set by default.

Using the ``domain-manager`` persona will be better than granting customers the
``admin`` role in their domain.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This work doesn't require any client code.

End users receiving the new ``domain-manager`` persona within a domain can
start using its permission set right away.

Performance Impact
------------------

This is a trivial change to add another RBAC-persona and corresponding
policy definitions. Performance impact is negligible.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

As this persona is intended for runtime usage, the only impact for developers
is to watch closely when adding new policies for the ``manager`` role, to avoid
accidently giving privileges to the domain-scoped ``manager``.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  None

Other contributors:
  None

Work Items
----------

* Adjust the default policies for group, project, user and role management
  actions in Keystone to incorporate the new domain-scoped ``manager`` in a way
  that enables management rights in domain scope only
* Add the ``domain-manager`` persona to the Keystone documentation describing
  its purpose and usage


Dependencies
============

None


Documentation Impact
====================

Any documentation that concerns identity management and Keystone usage should
include instructions for the ``domain-manager`` persona where appropriate.
This will be more or less restricted to Keystone since this persona is only of
use for the Identity API.


References
==========

* `Draft of the Domain Manager
  standard <https://github.com/SovereignCloudStack/standards/blob/main/Standards/scs-0302-v1-domain-manager-role.md>`_
  implementing the concept in Sovereign Cloud Stack using separate Keystone
  policy adjustments based on the current defaults
* `Etherpad notes from Caracal vPTG of Keystone discussing the topic and
  initiating this spec <https://etherpad.opendev.org/p/caracal-ptg-keystone>`_
* `Consistent and Secure RBAC: project manager role <https://governance.openstack.org/tc/goals/selected/consistent-and-secure-rbac.html#implement-support-for-project-manager-personas>`_
* `Launchpad bug: "admin"-ness not properly scoped <https://bugs.launchpad.net/keystone/+bug/968696>`_

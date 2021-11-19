..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Default Manager Role
====================

`bug #1951622 <https://bugs.launchpad.net/keystone/+bug/1951622>`_

In Rocky, keystone added a `default role hierarchy
<https://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/define-default-roles.html>`_.
This was part of a large initiative to improve RBAC across all OpenStack
projects. Through the process of adopting the default roles implemented in
Rocky, OpenStack developers and operators have acknowledged the need for
another default role that sits in between ``admin`` and ``member``.

This specification details the reason why we need another role in the
hierarchy, how we can extend ``keystone-manage bootstrap`` to implement it
cleanly, and how other OpenStack services can effectively use it.

Problem Description
===================

Today, ``keystone-manage bootstrap`` provides three default roles, ``admin``,
``member``, and ``reader``. These roles build a basic hierarchy, where
``admin`` implies ``member``, and ``member`` implies ``reader``. The rationale
for this work is detailed in the original specification.

In the process of adopting system-scope and default roles across OpenStack
projects, it's clear we need another role in the hierarchy.

Based on Yoga PTG `discussions
<https://etherpad.opendev.org/p/policy-popup-yoga-ptg>`_ about how we integrate
default roles and system-scope across all OpenStack components, developers and
operators decided that reserving the ``admin`` role for the highest level of
authorization on a given scope is a good idea. This means that operators would
never give their end users the ``admin`` role on a project. The highest level
of authorization they would allow users to have would be the ``member`` role on
a project.

This is fine for the majority of permissions an end user will need. For
example, create and deleting instances, volumes, snapshots, and networks within
a project. But, through the process of auditing OpenStack policies, we do see
the benefit of exposing some permissions to end users, but not every
project-member (e.g., anyone with the ``member`` role on a project.)

This is where we started to realize the need for another default role that sits
in above ``member`` and is designed to be given to end users.

The following are good examples of things a project-manager (e.g., anyone with
the ``manager`` role on a project) can do:

- Locking and unlocking an instance
- Sharing an image with other projects
- Setting the default volume type for a project
- Setting the default secret store for a project

Proposed Change
===============

Expand the ``keystone-manage bootstrap`` utility to create a new role named
``manager``. Conflicts should be handled gracefully, allowing an existing role
with that name to take precedence. Existing roles shouldn't be deleted and then
recreated, so that we don't break anything relying on the role ID.

Update the role implication so that the ``admin`` role implies ``manager`` and
the ``manager`` role implies ``member``. We should also update the role
implications so that ``admin`` no longer implies ``member`` directly, since
it's indirectly implied through the ``manager`` role.

OpenStack services looking to expose functionality to the project-manager
persona should write check strings as follows:

.. code-block:: python

   policy.DocumentedRuleDefault(
       name='foo',
       check_str='role:manager',
       scope_types=['project']
   )

Alternatives
------------

We could rely on deployment tooling to deploy this role at installation time.
However, that still opens the door for inconsistencies across OpenStack
installers.

Adding formal support in ``keystone-manage bootstrap`` is cleaner, more
consistent, and integrates with existing deployment tools without additional
action required.

Security Impact
---------------

This new role is not used by default across OpenStack policies, so it won't
carry any specific authorization until policies are updated to use it.

The only change operators will notice immediately is that they will
have an additional role called ``manager`` in their tokens. Granting someone
the ``manager`` role on a project won't have any impact initially and those
users will be limited to permissions accessible to project-members.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This work doesn't require any client code since it's done using
``keystone-manager``.

Performance Impact
------------------

This is a trivial change to add another row to the roles table and putting one
more role in the token response. Performance impact is negligible.

Other Deployer Impact
---------------------

If operators or deployers have a utility that creates a ``manager`` role, then
they can update that utility to remove that API call and they can rely on the
functionality in ``keystone-manage bootstrap``.

Additionally, if they have a use case for an authorization layer in between
``admin`` and ``member`` that we're trying to address with ``manager``, they
should being those use cases to the upstream policy popup team `meetings
<https://meetings.opendev.org/#Secure_Default_Policies_Popup-Team_Meeting>`_.

This will help developers understand which policies to consider applying the
``manager`` role to. It will also help operators converge their custom policy
onto the ``manager`` role and reduce the amount of custom overrides they need
in their deployment.

Developer Impact
----------------

Developers will have another role at their disposal for writing default
policies. They should be use to understand the ramifications of the ``manager``
role and ensure they're only using it with privileged end users in mind, at
least initially.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <lbragstad>

Work Items
----------

* Update ``keystone-manage bootstrap`` to create a new role called ``manager``
* Update role implications so ``manager`` is in the role hierarchy
* Add the corresponding manager personas (system-manager, domain-manager,
  project-manager) to the administrator `guide
  <https://docs.openstack.org/keystone/latest/admin/service-api-protection.html>`_
* Add the manager role to the developer `documentation
  <https://docs.openstack.org/keystone/latest/contributor/services.html#reusable-default-roles>`_
* Add the manager role to any OpenStack-wide documentation describing the
  secure RBAC personas

Dependencies
============

This work is required to move forward on a set of community-wide `goals
<https://review.opendev.org/c/openstack/governance/+/815158>`_ to improve
authorization in OpenStack.

Documentation Impact
====================

Listed above in the `Work Items`_ section.

References
==========

Referenced inline.

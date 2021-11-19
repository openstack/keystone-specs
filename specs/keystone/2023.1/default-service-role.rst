..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Default Service Role
====================

`bug #1951632 <https://bugs.launchpad.net/keystone/+bug/1951632>`_

In Rocky, keystone added a `default role hierarchy
<https://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/define-default-roles.html>`_.
This was part of a large initiative to improve RBAC across all OpenStack
projects. Through the process of adopting the default roles implemented in
Rocky, OpenStack developers and operators have acknowledged that several
OpenStack service accounts have too much authorization.

This specification details the reasons for creating a new role called
``service`` and updating the default policies across OpenStack for
service-to-service communication to use that specialized role.

Problem Description
===================

Today, ``keystone-manage bootstrap`` provides three default roles, ``admin``,
``member``, and ``reader``. These roles build a basic hierarchy, where
``admin`` implies ``member``, and ``member`` implies ``reader``. The rationale
for this work is detailed in the original specification.

The ``admin`` role has the ability to do just about anything in the OpenStack
deployment. Unfortunately, it's used as catch all for authorization. This
results in service users getting the ``admin`` role when they need to do a
privileged operation in one service. In reality, this is poor security practice
because that service account has the ability to do anything. It can create,
update, and remove any resource in the deployment. A bad actor could easily use
a compromised service account to create backdoor access to the deployment.

By applying the principle of least privilege to service users, we can harden
OpenStack default security posture and reduce the surface area of
administrative power.

Proposed Change
===============

Expand the ``keystone-manage bootstrap`` utility to create a new role named
``service``. Conflicts should be handled gracefully, especially since some
policies in OpenStack rely on the ``service`` role already and it's entirely
plausible that operators have created this role already. Existing roles
shouldn't be deleted and then recreated, so that we don't break anything
relying on the role ID.


This role should be kept outside of the existing role hierarchy that includes
``admin``, ``member``, and ``reader``. Those role definitions were created with
humans in mind, and organized hierarchically to apply to hierarchical RBAC.
Service-to-service communication is much more prescriptive in that service
accounts only need to access a handful of APIs from other services. Building
the ``service`` role outside the current hierarchy ensures we're following the
principle of least privilege for service accounts.

After keystone implements support for the ``service`` role, OpenStack service
developers can start integrating it into their default policies.

.. code-block:: python

   policy.DocumentedRuleDefault(
       name='os_compute_api:os-server-external-events:create',
       check_str='role:service',
       scope_types=['project']
   )

We need to keep all the service-to-service APIs default to ``service`` role
only and not to add any other role in logical or here. The idea here is to keep
their default access to ``service`` role only. If any of the service-to-service
APIs are used by admin or non-admin user then recommendation is to ask them to
override the default in policy.yaml file instead of changing the default in
code. There might be exception service-to-service APIs which project think are
useful to be used by admin or non-admin user then they can take the exceptional
decision to default them to user role and ``service`` role.

As we have dropped the system scope implementation for services, service to
service communication with ``service`` role will be done with project scope
token.

Additionally, any deployment tools that create service accounts for OpenStack
services, should start preparing for these policy changes by updating their
role assignments and performing the deployment language equivalent of the
following::

   $ openstack role add --user nova --project service service
   $ openstack role add --user cinder --project service service
   $ openstack role add --user neutron --project service service
   $ openstack role add --user glance  --project service service
   $ openstack role add --user manila  --project service service

Future Work Proposals
---------------------

Service-Specific Default Roles
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In a later release we could elaborate on this concept to create additional
roles called ``compute-service``, ``block-storage-service``,
``network-service``, ``image-service`` and so on.  The ``service`` role could
imply each of these roles and the policies for each service could be refined
again to only allow access based on the service. This would require additional
deployment tool intervention::

   $ openstack role add --user nova --project service compute-service
   $ openstack role add --user cinder --project service block-storage-service
   $ openstack role add --user neutron --project service network-service
   $ openstack role add --user glance --project service image-service

This would reduce the ability for service accounts to interact with APIs they
don't need.

Application Credentials
^^^^^^^^^^^^^^^^^^^^^^^

An alternative to service-specific default roles would be to use `application
credentials with access rules
<https://docs.openstack.org/keystone/latest/user/application_credentials.html#access-rules>`_
for each service. Each service would still have the ``service`` role, but it
would be restricted to only access the APIs it requires using access rules.

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

This new role is already used in some OpenStack policies. Using it initially
will be better than granting service users the ``admin`` role.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This work doesn't require any client code since it's done using
``keystone-manager``.

Performance Impact
------------------

This is a trivial change to add another row to the roles table. Performance
impact is negligible.

Other Deployer Impact
---------------------

If operators or deployers have a utility that creates a ``service`` role, then
they can update that utility to remove that API call and they can rely on the
functionality in ``keystone-manage bootstrap``.

Developer Impact
----------------

Developers will have another role at their disposal for writing default
policies. They should understand the proper usage of ``service`` and that it
should only be used for service-to-service APIs.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <lbragstad>
  <abhishekk>

Work Items
----------

* Update ``keystone-manage bootstrap`` to create a new role called ``service``
* Add the corresponding ``service`` role to the administrator `guide
  <https://docs.openstack.org/keystone/latest/admin/service-api-protection.html>`_
* Add the ``service`` role to the developer `documentation
  <https://docs.openstack.org/keystone/latest/contributor/services.html#reusable-default-roles>`_
* Add the service role to any OpenStack-wide documentation describing the
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

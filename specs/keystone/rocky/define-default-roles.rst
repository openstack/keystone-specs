===================
Basic Default Roles
===================

`blueprint basic-default-roles <https://blueprints.launchpad.net/keystone/+spec/basic-default-roles>`_

Managing `Role Based Access Control
<https://csrc.nist.gov/Projects/Role-Based-Access-Control>`_ (RBAC) across
OpenStack is one of the hardest pain points for operators to deal with. It is
not uncommon for operators to have to dig through source code and keep notes
about oddities in RBAC implementations across OpenStack just to offer basic
RBAC capabilities to their customers. End users are also affected because it is
very rare for any two deployments to have similar roles, or ensure those roles
are mapped to similar operations.

Problem description
===================

OpenStack's initial implementation of RBAC was simple and worked for trivial
deployments. As OpenStack evolved and deployments started modeling larger, more
complex organizations, the RBAC implementation failed to evolve with it. As a
result, operators are stuck using existing tooling to provide the facade of a
more sophisticated RBAC solution. This is a confusing and incredibly tough
maintenance burden for operators who customize policy.

It's not uncommon to see various services hardcode operations to a specific
role. While the operation may require that role, the role to policy mapping
should be driven by policy defaults that can be overridden by operators instead
of hardcoding.

Proposed change
===============

As a platform, OpenStack should offer a basic, easy to understand RBAC
implementation with clear, reasonable default values. The process of
implementing this will give operators more flexibility out-of-the-box. It will
also be less likely to introduce inconsistencies across deployments due to the
limitations of the existing implementation.

To help ensure a graceful transition, `improvements
<http://specs.openstack.org/openstack/oslo-specs/specs/queens/policy-deprecation.html>`_
were made to the oslo policy library and a community `goal
<https://governance.openstack.org/tc/goals/queens/policy-in-code.html>`_ put in
place to help projects teams register defaults policies in code and provide
documentation. This work gives OpenStack project teams the tools necessary to
improve default role definitions. The changing defaults can be consumed by
operators in ways that are consistent with changing configuration options.

This specification proposes that Keystone enhance the basic RBAC experience
by incorporating the following default roles into its default policies.

The work detailed here can be separated into two initiatives. The first is
ensuring the defaults proposed are available to operators after installation.
The second is incorporating those available roles into default policies across
services. Note that the first initiative was targeted and completed in the
Rocky release. While this specification does go into detail describing the
second initiative, it will be implemented in a subsequent release (likely Stein
or later). The second initiative specifically within keystone will require
landing a large refactor cleaning up technical debt and moving keystone to
using `flask <https://bugs.launchpad.net/keystone/+bug/1776504>`_ instead of a
home-grown WSGI implementation. It is imperative to land this refactor prior to
starting the second initiative because it will make treating RBAC across
different scopes like formal business logic across the Manager layers within
keystone subsystems, as opposed to obfuscating more complexity into the
``@controller.protected`` decorator that is currently used by most APIs.

Our goal is that this work will serve as a template which other services may
use to adopt the proposed default roles in a future `community goal
<https://governance.openstack.org/tc/goals/>`_.

Default Roles
-------------

**reader**: It should only be used for read-only APIs and operations.
Alternatively referred to as ``readonly`` or ``observer``, this role fills
an extremely popular need from operators.

**member**: serves as the
general purpose ‘do-er’ role. It introduces granularity between
the administrator(s) and everyone else.

**admin**: This role will be only be considered appropriate for operations
deemed too sensitive for anyone with a member role.

The desired outcome of implementing the roles above is that projects should
start moving away from the practice of hardcoding operations to specific role
names. Instead, each policy should have a reasonable default that can be
overridden by operators.

Scope Type (Refresher)
----------------------

**project-scope**: Project-scope relates to authorization for operating in a
specific tenancy of the cloud.

**system-scope**: System-scope relates to authorization for operating with APIs
that do not map nicely to the concept of Project scope. It is **not** meant to
cover *all* APIs across a deployment. More information about system-scope can
be found in the `specification <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/system-scope.html>`_,
along with relevant historical context justifying the `need for system-scope
<https://bugs.launchpad.net/keystone/+bug/968696>`_.

Examples
--------

`reader:`
An example project-scoped application of this role would be listing project
tags (``identity:get_project_tags``).
An example system-scoped application of this role would be listing service
endpoints (``identity:list_endpoints``).

`member:`
An example project-scoped application of this role would be creating a project
tag (``identity:update_project_tags``).
An example system-scope application of this role would be updating an endpoint
(``identity:update_endpoint``).

`admin:`
An example project-scoped administrator operation would be deleting project
tags (``identity:delete_project_tags``).
An example system-scoped administrator operation would be creating an endpoint
for a service (``identity:create_endpoint``) or
listing migrations (``os_compute_api:os-migrations``).


The following is neither a final nor a comprehensive list of all possible
rules/policies. It serves merely as a snippet of existing rules to showcase how
policies, scope, and the new default roles can work together to provide a
richer policy experience.

Project Reader
~~~~~~~~~~~~~~

* ``identity:list_project_tags``
* ``identity:get_project_tag``

Project Member
~~~~~~~~~~~~~~

* ``identity:list_project_tags``
* ``identity:get_project_tag``
* ``identity:update_project_tags``

Project Admin
~~~~~~~~~~~~~

* ``identity:list_project_tags``
* ``identity:get_project_tag``
* ``identity:update_project_tags``
* ``identity:create_project_tags``
* ``identity:delete_project_tags``

System Reader
~~~~~~~~~~~~~

* ``identity:list_endpoints``
* ``identity:get_endpoint``

System Member
~~~~~~~~~~~~~

* ``identity:list_endpoints``
* ``identity:get_endpoint``
* ``identity:update_endpoints``

System Admin
~~~~~~~~~~~~

* ``identity:list_endpoints``
* ``identity:get_endpoint``
* ``identity:update_endpoints``
* ``identity:create_endpoints``
* ``os-compute-api:os-hypervisors``
* ``os-compute-api:os-migrations``

Example snippets of various policy files, or rendered snippets, could look like
the following.

.. note::

  The default roles discussed will be created by Keystone, during the bootstrap
  process, using `implied roles
  <https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/implied_role.html>`_.
  As indicated in the above table, having ``admin`` role implies a user also
  has the same rights as the ``member`` role. Therefore this user will also has
  the same rights as the ``reader`` role as ``member`` implies ``reader``.

  This keeps policy files clean. For example, the following are equivalent as a
  result of implied roles:

  "identity:list_endpoints": "role:reader OR role:member OR role:admin"
  "identity:list_endpoints": "role:reader"

   The chain of implied roles will be documented alongside of the
   `policy-in-code defaults
   <https://github.com/openstack/keystone/blob/master/keystone/common/policies/base.py>`_
   in addition to general Keystone documentation updates noting as much.

::

    # scope_types = ('project')
    "identity:list_project_tags": "role:reader"
    "identity:get_project_tag": "role:reader"
    "identity:update_project_tags": "role:member"
    "identity:create_project_tag": "role:admin"
    "identity:delete_project_tags": "role:admin"

    # scope_types = ('system')
    "identity:list_endpoints": "role:reader"
    "identity:get_endpoints": "role:reader"
    "identity:update_endpoint": "role:member"
    "identity:create_endpoint": "role:admin"
    "os_compute_api:os-hypervisors": "role:admin"
    "os_compute_api:os-migrations": "role:admin"


Let's assume the following role assignment exist:

- **Alice** has role **reader** on system
- **Bob** has the role **member** on system
- **Charlie** has role **admin** on system
- **Qiana** has role **reader** on Project Alpha
- **Rebecca** has role **member** on Project Alpha
- **Steve** has role **admin** on Project Alpha

Given the above assignments and policies, the following would be possible:

**Alice** can list or retrieve specific endpoints. Alice cannot do any project
specific operations since her authorization is limited to the deployment
system.

**Bob** can retrieve specific endpoints, list them, and update them. He cannot
create new endpoints, or delete existing ones. Bob cannot do any project
specific operations because his authorization is limited to the deployment
system.

**Charlie** can retrieve specific endpoints, list, as well as create them.
Additionally, Charlie can list information on migrations as well as
hypervisors. He cannot perform any project specific operations because his
authorization is limited to the deployment system.

**Qiana** can list all tags and get details about a specific tag within Project
Alpha. She may not perform system specific policies because her authorization
is on a single project.

**Rebecca** can list all tags, get details about a specific tag, and update a
tag within Project Alpha. She cannot perform any system specific policies
because her authorization is on a single project.

**Steve** can list all tags, create new tags, get details about a specific tag,
update a tag, and delete tags within Project Alpha. He cannot perform any
system specific policies because his authorization is on a single project.

Risk Mitigation
---------------

**Scenario One -- A role serving the purposes described in this spec exists
under another name**: Let us assume that Deployment A already has ``Role X``
which serves the purpose of the proposed here as the ``reader`` role. In this
instance, it is reasonable to assume that operators may have custom policy work
in place and do not want to port immediately.

This issue may be mitigated through the use of implied roles. Operators need
simply to ensure that ``reader`` implies ``Role X``. Please review the
documentation on `implied roles
<https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/implied_role.html>`_.
for specific instructions on how make one role imply another.

**Scenario Two -- An existing ``reader``, ``member``, or ``admin`` role already
exists**: Let us assume that Deployment B already has a ``member`` role.
Keystone will not attempt to overwrite any existing roles that have been
populated. It will instead note that a role with the name ``member`` already
exists in log output. However, the role implications *will* still be created
regardless of whether the role existed previously or not.

Alternatives
------------

reader/writer/admin vs reader/member/admin. There was much debate regarding the
naming conventions for these roles. We have opted to use `reader`, `member`,
and `admin` as we believe they most accurately describe their purpose when the
context of OpenStack is taken into consideration.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Lance Bragstad lbragstad lbragstad@gmail.com
* Harry Rybacki hrybacki hrybacki@redhat.com

Work Items
----------

* Add ability for Keystone bootstrap to create proposed roles
* Implement reader role across policies
* Implement member role across policies
* Implement admin role across policies
* Implement scope_types for all policies in Keystone
* Remove @protected decorator
* Document how operators may generate policy files with service specific roles
* Prepare Proof-of-Concept to demo and facilitate acceptance of an OpenStack
  Community Goal to promote default roles across the other services.

Dependencies
============

This work is dependent on the following:

* `Registering and documenting
  <https://governance.openstack.org/tc/goals/queens/policy-in-code.html>`_
  all policies in code

* `Use flask <https://bugs.launchpad.net/keystone/+bug/1776504>`_

The work detailed in this specification will be supplemented with policy work
being done in oslo and keystone:

* Implementing `system-scope
  <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/system-scope.html>`_
  in keystone
* Implementing `scope_types
  <http://specs.openstack.org/openstack/oslo-specs/specs/queens/include-scope-in-policy.html>`_

Full dependencies and relevant work can be found in the `Policy Roadmap
<https://trello.com/b/bpWycnwa/policy-roadmap>`_.

Resources
=========

* `Policy Roadmap <https://trello.com/b/bpWycnwa/policy-roadmap>`_
* `System Scope <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/system-scope.html>`_
* `Deprecation with oslo.policy <http://specs.openstack.org/openstack/oslo-specs/specs/queens/policy-deprecation.html>`_
* `Scope types in oslo.policy <http://specs.openstack.org/openstack/oslo-specs/specs/queens/include-scope-in-policy.html>`_
* Previous `attempts <https://review.openstack.org/#/c/245629>`_ at providing
  default roles


.. note::

  This work is licensed under a Creative Commons Attribution 3.0 Unported
  License.  http://creativecommons.org/licenses/by/3.0/legalcode


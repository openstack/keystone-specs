..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
System Role Assignments
=======================

`bp system-scope <https://blueprints.launchpad.net/keystone/+spec/system-scope>`_

This document describes the necessary changes in order to implement system role
assignments.

Problem Description
===================

Today, role assignments are built by giving an actor a role on a target. An
actor can be a user or a group. A target is limited to a project or a domain.
This works great for controlling access to things that map into a project or
domain (e.g. instance ownership fits naturally within projects). This starts to
get confusing with operations that don't fit within that constraint. Performing
operations on hypervisors (e.g. `GET /os-hypervisors/`) is a good example of an
API that doesn't map well to a project. Instead, it's clearer to think about
these types of operations at a system-wide perspective, instead of a
project-specific one.

A system role assignment would be an assignment with the target being `system`
instead of a project or a domain.

Global Scope vs. System Scope
-----------------------------

Initial discussions around this proposal used the term *global scope* as a way
to distinguish something other than project scope. For example, operations on
instances are easy to associate to a project because instances are owned by
projects. Endpoints or services were originally considered *global* in nature
since they applied to the whole deployment. After several discussions, it
became apparent that *global* still wasn't the term we were looking for.
Another example highlights the deficiency in the term *global*. Theoretically,
if a user has a role assignment on the root domain, or project, in an
deployment, shouldn't they be able to view all instances in the deployment
(e.g. the entire project tree under the root domain)? Is that not in some sense
also *global* because the user would be viewing all instances across the entire
deployment? After understanding that, it became apparent that *global* refers
to the root of a tree, when we really needed it to refer to *system*
operations.

Using the term *system* helps clarify which resources and APIs are specific to
the function of the deployment, or system as a whole. For example, service and
endpoints are entities required for the *system* to function properly. They
clearly don't pertain to a single project or domain. Hypervisor management in
nova is also a *system* level resource that doesn't make sense to associate to
a single project. Multiple projects can have instances hosted on a single
hypervisor due to multi-tenancy.

Proposed Change
===============

List system role assignments for a user
---------------------------------------

**Request:** `GET /v3/system/users/{user_id}/roles/`

**Parameters**

* `user_id` - The user ID.

**Response**

* 200 - OK
* 404 - Not Found if a user doesn't exist
* 401 - If the operation isn't permitted to the user

**Response Body**

.. code:: json

    {
        "links": {
            "self": "http://example.com/identity/v3/system/users/cf8e3ee7115b4a88897673ee61dd2919/roles",
            "previous": null,
            "next": null
        },
        "roles": [
            {
                "id": "46b213b41e7344cc8078ac5e7d161f17",
                "links": {
                    "self": "http://example.com/identity/v3/roles/46b213b41e7344cc8078ac5e7d161f17"
                },
                "name": "admin"
            }
        ]
    }

Assign a system role to a user
------------------------------

**Request:** `PUT /v3/system/users/{user_id}/roles/{role_id}`

**Parameters**

* `user_id` - The user ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or user doesn't exist
* 401 - If the operation isn't permitted to the user

Check if a user has a system role assignment
--------------------------------------------

**Request:** `HEAD /v3/system/users/{user_id}/roles/{role_id}`

**Request:** `GET /v3/system/users/{user_id}/roles/{role_id}`

**Parameters**

* `user_id` - The user ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or user doesn't exist
* 401 - If the operation isn't permitted to the user

Unassign a system role from a user
----------------------------------

**Request:** `DELETE /v3/system/users/{user_id}/roles/{role_id}`

**Parameters**

* `user_id` - The user ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or user doesn't exist
* 401 - If the operation isn't permitted to the user

List system role assignments for a group
----------------------------------------

**Request:** `GET /v3/system/groups/{group_id}/roles/`

**Parameters**

* `group_id` - The group ID.

**Response**

* 200 - OK
* 404 - Not Found if a group doesn't exist
* 401 - If the operation isn't permitted to the user

**Response Body**

.. code:: json

    {
        "links": {
            "self": "http://example.com/identity/v3/system/groups/282051ffddcf4206a954ad838c86d39f/roles",
            "previous": null,
            "next": null
        },
        "roles": [
            {
                "id": "46b213b41e7344cc8078ac5e7d161f17",
                "links": {
                    "self": "http://example.com/identity/v3/roles/46b213b41e7344cc8078ac5e7d161f17"
                },
                "name": "admin"
            }
        ]
    }

Assign a system role to a group
-------------------------------

**Request:** `PUT /v3/system/groups/{group_id}/roles/{role_id}`

**Parameters**

* `group_id` - The group ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or group doesn't exist
* 401 - If the operation isn't permitted to the user

Check if a group has a system role assignment
---------------------------------------------

**Request:** `HEAD /v3/system/groups/{group_id}/roles/{role_id}`

**Request:** `GET /v3/system/groups/{group_id}/roles/{role_id}`

**Parameters**

* `group_id` - The group ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or group doesn't exist
* 401 - If the operation isn't permitted to the user

Unassign a system role from a group
-----------------------------------

**Request:** `DELETE /v3/system/groups/{group_id}/roles/{role_id}`

**Parameters**

* `group_id` - The group ID.
* `role_id` - The role ID.

**Response**

* 204 - No Content
* 404 - Not Found if a role or group doesn't exist
* 401 - If the operation isn't permitted to the user

List role assignments
---------------------

The existing API to list role assignment will have to be enhanced to return
system role assignment, in addition to the project and domain role assignment
information it returns today.

**Request:** `GET /v3/role_assignments`

**Paramters**

A filter will be added, called `scope.system`, to filter role assignments by
system-specific role assignment. It will be a boolean value.

**Response**

* 200 - OK
* 401 - If the operation isn't permitted to the user

**Response Body**

.. code:: json

    {
        "role_assignments": [
            {
                "role": {
                    "id": "d6c89e9121304b6f87de57b0500b0526"
                },
                "user": {
                    "id": "3f0c5f11e792494ab5de347696fa1421"
                },
                "scope": {
                    "domain": {
                        "id": "6bfbd79b010e4405b92731479cbbe8e7"
                    }
                },
                "links": {
                    "assignment": "http://example.com/identity/v3/domains/6bfbd79b010e4405b92731479cbbe8e7/users/3f0c5f11e792494ab5de347696fa1421/roles/d6c89e9121304b6f87de57b0500b0526"
                }
            },
            {
                "role": {
                    "id": "2fb8d689a8744a42af926ea4f8f929c7"
                },
                "group": {
                    "id": "a806d9029db7403e9869632aee082e5c"
                },
                "scope": {
                    "project": {
                        "id": "2fae742cb86543af825471ea6b63ccea"
                    }
                },
                "links": {
                    "assignment": "http://example.com/identity/v3/projects/2fae742cb86543af825471ea6b63ccea/groups/a806d9029db7403e9869632aee082e5c/roles/2fb8d689a8744a42af926ea4f8f929c7"
                }
            },
            {
                "group": {
                    "id": "1d8d919f37d94f308d007e72737cf10a"
                },
                "links": {
                    "assignment": "http://example.com/identity/v3/system/groups/1d8d919f37d94f308d007e72737cf10a/roles/b29d6fff51c43478b00bb16bfb771fc"
                },
                "role": {
                    "id": "ab29d6fff51c43478b00bb16bfb771fc"
                },
                "scope": {
                    "system": true
                }
            }
        ],
        "links": {
            "self": "http://example.com/identity/v3/role_assignments",
            "previous": null,
            "next": null
        }
    }

Authenticating for a system-scoped token
------------------------------------------

The following is an example request for a system-scoped token::


    {
        "auth": {
            "identity": {
                "methods": [
                    "password"
                ],
                "password": {
                    "user": {
                        "id": "8bbca32b850a4c22b64a1b7bc2c6bd13",
                        "password": "my-password"
                    }
                }
            },
            "scope": {
                "system": {
                    "all": true
                }
            }
        }
    }

An example response would be::

    {
        "token": {
            "audit_ids": [
                "doIh18J8RyW3jXF50FV26g"
            ],
            "catalog": [
                ...
            ],
            "expires_at": "2017-05-15T21:58:29.000000Z",
            "issued_at": "2017-05-15T20:58:29.000000Z",
            "methods": [
                "password"
            ],
            "system": {
                "all": true
            },
            "roles": [
                {
                    "id": "c2145c84a802413fbac71479250c9378",
                    "name": "observer"
                },
                {
                    "id": "fc2ec22e227941f8afd94a1587ac57d3",
                    "name": "admin"
                }
            ],
            "user": {
                "domain": {
                    "id": "default",
                    "name": "Default"
                },
                "id": "8bbca32b850a4c22b64a1b7bc2c6bd13",
                "name": "bob",
                "password_expires_at": null
            }
        }
    }

System scope can be consumed by existing policies::

    "system_admin": "role:admin and system:True"
    "system_reader": "role:reader and system:True"
    "admin_required": "rule:system_admin"

The attributes of a system token response can also be consumed by
`oslo.context` and exposed to services for scope checks using `context.scope =
'system'` or some other method. The process of relaying this information to the
consuming service will contain follow on work to the `oslo.context` library to
ensure it handles system-scoped tokens properly. The primary purpose of this
specification is to allow for the scoping of roles at a system level and
exposing that ability to end users. Work can be done in parallel to consume
this information in policy files or shared libraries.

System Roles, Implied Roles, & Inherited Roles
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Keystone supports other types of role behaviors. An administrator can have one
role imply another, or have roles be inherited according to the hierarchical
structure of projects. For example, if role ``Alpha`` implies role ``Beta``, a
user with role ``Alpha`` will automatically be given role ``Beta`` on the
target, since it's implied.  Another example is if a role assignment is allowed
to be inherited through a tree of projects. For example, if ``project C`` is
the parent of ``project D`` and a user has role ``Echo`` on  ``project C``, the
user also has role ``Echo`` on ``project D`` via role inheritance.  These
concepts are known respectively, as implied roles and inherited roles.

Part of introducing a system scoping mechanism is understanding how it applies
to these concepts. It is possible to apply both of these concepts to system
roles. A role assigned to a user on the system should be able to imply other
roles. There have been discussions about making the system a hierarchy
structure in the future. For example, what if a system was actually a tree of
regions. That would introduce another level of scope that allows users to have
role assignments on subsets of the entire system. This seems like a powerful
idea, but it does need more thought and discussion. For the time being, system
will be a single entity, but built to be refactored into a hierarchy later.

In conclusion, the initial implementation of system roles should support
implied role assignment. It should be flexible enough to support inherited
roles if the system entity ever evolves into a tree of regions or services.

Alternatives
------------

An alternative to this approach would be to leverage the `admin_project` in
order to achieve global scoping. The `admin_project` is a special project that
allows for elevated privileges if role assignments are given to that project.
Let's consider the following example. Let's say there is an `observer` role
that allows users to perform read-only operations within a specific scope.
If Bob has the `observer` role on project `foo`, he should be able to view
things within that project. If Alice has the `observer` role on the
`admin_project`, she should be able to view things across the deployment, like
services and endpoints.

In this model, system scope is determined by a specific project and the role
assignments that project has. Every user that requires a system role (i.e.
admin, observer, support, etc) in a deployment will be required to have a role
assignment on the `admin_project`.

Benefits:

* Reuse of existing project scope mechanisms/tokens
* Leveraging the `is_admin_project` attribute of tokens
* Most of this work is already done
* Not necessary to change how scope is stored

Drawbacks:

* Automated tooling might have to handle this project separately (i.e. coding
  around an implementation detail of how policy is elevated) to ensure nothing
  happens to the `admin_project`
* Operators may find it confusing to have a role on a super-special project in
  order to have elevated privileges, which seems like an anti-pattern
* All users that require a system role of some kind must have a role assignment
  on the `admin_project`, this could result in a large number of role
  assignments on the `admin_project`
* Develop some sort of recovery plan in the event the `admin_project` is
  accidentally deleted
* Certain resources can't belong in system scope today (i.e. instances must be
  tied to a project), this approach doesn't stop users from creating resources
  within the `admin_project`, which would be the equivalent to a system-wide
  instance
* How does the `admin_project` conform to project hierarchy? Is it suppose to
  be kept in it's own subtree under the default domain or can it have child
  projects underneath it?

Roadmap
-------

The `is_admin_project` implementation exists in OpenStack today, is relayed
through keystone APIs, and present in some service policy files. It makes sense
to have compatibility for both moving forward. The `roadmap <https://etherpad.openstack.org/p/queens-PTG-keystone-policy-roadmap>`_
put together at the Queens PTG shows how we can improve admin-ness using both
approaches but end up in a place where system scope is required.

Security Impact
---------------

This type of scoping will allow OpenStack services to separate system
operations from project or domain scoped operations. The result will be an
improved security model across OpenStack. Note that a system-scoped token is
still a bearer token and allows the holder the ability to do things on the
deployment system.

Notifications Impact
--------------------

System scoping will be subject to the same notifications as project or domain
scope requests.

Other End User Impact
---------------------

This is highly dependent on how operators have configured their policy across
OpenStack. Ideally, this will give operators more tools to provide better
security in their deployments.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

Deployers will now have the ability to control system operations by leveraging
system role assignments. The ability will be available by default but a
migration won't be supplied to migrate existing policy workarounds since policy
can vary wildly across deployments.

An upgrade document can be provided to help operators visualize the process and
apply it to their specific policy scenario.

Developer Impact
----------------

This work will most-likely require some changes to testing both inside and
outside of keystone, in order to guarantee isolation of system operations from
project operations. Mitigating this will be a required work item of the
implementation.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Lance Bragstad <lbragstad@gmail.com> lbragstad

Other contributors:
  None

Work Items
----------

* Add a new database table to support system assignments
* Implement system role assignments
* Implement scoping a token to a system context
* Migrate tempest testing to leverage system roles
* Clearly document possible upgrade paths for operators
* Implement system context in `oslo.policy` and `keystonemiddleware`

Follow on work items should be done to ensure system role assignments are
honored within policies across OpenStack:

* Ensure default policies adhere to system scope
* Ensure scope checks across projects enforce system scope


Dependencies
============

None.

Documentation Impact
====================

We will need to provide a more consistent authentication document that clearly
explains scope at the project and system level. A separate document that
describes possible upgrade paths from the existing system will also be a
requirement.

References
==========

None.

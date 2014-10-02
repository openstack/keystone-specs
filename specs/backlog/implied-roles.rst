..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Implied Roles - Assign one Role, get many
=========================================

`bp implied-roles <https://blueprints.launchpad.net/keystone/+spec/implied-roles>`_

Allow role definitions to imply other role definitions, so that, with one
superior role assignment, the user will inherit all subordinate roles.



Problem Description
===================

OpenStack is attempting to solve problems of scale.  The RBAC model
needs to support improved delegation in order to scale.

Keystone roles map to a set of permissions to perform operations on
resources. Today, there are relatively few roles defined, with each
being mapped to a large number of APIs.  This is very coarse grained
access control.  If deployers are going to provide a finer-grained
model, the number of roles will quickly escalate.  To minimize the
burden on the  adminstrators, it must be possible to grant a user all
the privileges they need for a specific project with a single role
assignment.  This is not possible with the current role assignment
mechanism.



As an organization increases in size, the amount of hierarchy must
increase accordingly, or managers quickly become overwhelmed by the
sheer number of requests by subordinates.  A manager has
responsibility for all of the operations performed by a subordinate,
even if they never choose to perform those operations directly.  The
authority to perform those operations comes from the superior to the
subordinate.   If a superior cannot delegate appropriately, an
organization cannot function.

When interacting with automated systems, a user should be able to
delegate a subset of their capabilities.  If a user is a system
administrator, they are irresponsible delegating full administration
to a user when the required operation is not a privileged operation
or even if it affects only a subset of resources.  Even casual users
should be able to provide access to a subset of their resources
without providing full access to all of them.  In order to split apart
permissions this way, the roles need to provide sufficient information
to deduce that a larger and more powerful role implies a smaller and
more limited role.  This is also not possible with the current role
assignment mechanism.


Proposed Change
===============


Users are assigned to roles within a project to perform
operations.  In order to better model the typical hierarchical
authority model of a large organization, we will allow relationships between
roles to be defined where one (larger more powerful role) can imply
that the user also obtains a set of smaller (less powerful) roles.

Often, those operations must be performed by users with
different roles.  As such, the roles can be viewed as a hierarchy;
larger roles inherit the permissions assigned to smaller roles.  In
order to implement this hierarchy, the relationship between the roles
must clearly indicate when a prior role implies another role.

In a persisted store, define a set of rules that state one prior role
assignment implies additional role assignments.

For example, if a rule states that `admin` implies `member` any user
assigned the `admin` role will automatically receive the `member` role
as well.

First implementation is the SQL back-end.  Other
back-ends will follow if requested. Add an additional implied_role
table with two columns:

prior_role_id,  implied_role_id

This is often termed hierarchical roles, but this implementation avoids a
strict hierarchy in favor of generating a directed-acyclic-graph (DAG): the
same role may be implied by multiple prior roles.  At enforcement time
the required abstraction is a set of role assignments, not a tree or
a graph.


An example set of implied roles:

The `reader` role is for people that need to be able to inspect the values
of resources in a project, but not make any changes to those resources.

The `editor` role is for people that need to make standard changes, such as
creating new virtual machines and  allocating floating IP address.  All
`editors`  should have access to the resources specified by the `reader`
role.

Each of the services have their own admin roles defined.  In addition, the two
storage focused services have a joint role called `storage_admin` that implies
both `swift_admin` and `cinder_admin`.

If a user is assigned `all_admin`  in a project and requests a token for that
project, the token will have all of the implied roles enumerated in it:
`cinder_admin`,  `cinder_admin`, `neutron_admin`, `glance_admin`,
`swift_admin`, `editor`, and `reader`.

Any form of admin is implicitly an `editor`. A `reader` can view standard
data from any of the systems, but cannot affect any change.   The `editor`
role is superior to `reader`:  there would be no reason to assign someone the
`editor` role without assigning the `reader` role as well.  We often want to
assign the `reader` role without the `editor` role for audit and monitoring.


The role relationships are illustrated in this directed acyclic graph
(DAG) diagram.  The prior roles are above implied roles, with the
arrows showing the direction of implication.  The table below also
explicitly shows these relationships


::

                   all_admin
                         |
                         V
           +-------------+-------------+---------+----------+
           |             |             |         |          |
           |             |             |         V          |
           |             |             |     storage_admin  |
           |             |             |         +          |
           |             |             |   +-----+------+   |
           V             V             V   V            V   V
      neutron_admin glance_admin swift_admin          cinder_admin
           |             |             |                    |
           +-------------+-------+-----+--------------------+
                                 |
                                 V
                              editor
                                 |
                                 V
                              reader


Note it is not a strict hierarchy.  For example,  both the `neutron_admin` and
the `glance_admin` roles imply the editor role.


Here is an example implementation

+=================+======================+
| prior_role_id   |  implied_role_id     |
+=================+======================+
| all_admin       | neutron_admin        |
+-----------------+----------------------+
| all_admin       | glance_admin         |
+-----------------+----------------------+
| all_admin       | swift_admin          |
+-----------------+----------------------+
| all_admin       | cinder_admin         |
+-----------------+----------------------+
| all_admin       | storage_admin        |
+-----------------+----------------------+
| storage_admin   | swift_admin          |
+-----------------+----------------------+
| storage_admin   | cinder_admin         |
+-----------------+----------------------+
| neutron_admin   | editor               |
+-----------------+----------------------+
| glance_admin    | editor               |
+-----------------+----------------------+
| swift_admin     | editor               |
+-----------------+----------------------+
| cinder_admin    | editor               |
+-----------------+----------------------+
| editor          | reader               |
+-----------------+----------------------+


Both explicitly assigned and implied roles will be included in the token
validation response.  With the above example, if a user was explicitly
assigned the role `editor` on a project, the validation of a token for
that user and scoped to the project would have the roles:  `editor`
and `reader` included in the response.


An initial configuration option of infer_roles' in the [token]
section of the config file will control whether to expand roles when
issuing tokens.


Alternatives
------------

Dispense with role hierarchies by simply assigning a user to the superior roles
and all the subordinate roles. Then he inherits all the privileges assigned to
all the roles. The advantage of role hierarchies is that the user does not need
to carry all the subordinate roles around with himself as the system knows the
role hierarchy.

Role implication rules can be fetched separately from the token,
cached in auth-token middleware, and then roles can be inferred from
the token prior to policy enforcement.  This will be implemented if
required.

A dynamic policy mechanism can use the implied roles to generate a
section of the policy files.



Security Impact
---------------


* Does this change touch sensitive data such as tokens, keys, or user data?
* Yes:  The token creation process will now be adding more roles on to a token,
  especially for roles high in the hierarchy.  The ability to create
  implied roles is a very sensitive ability and should be tightly controlled.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
* Yes.  Role assignments now may have associated implicit assignments.



Notifications Impact
--------------------

One notification will be sent out on each change of the implied roles
rules.


Other End User Impact
---------------------

Once this change is in place, role checks in policy files should be
streamlined to check a smaller set of potential roles.


Performance Impact
------------------

Token validation responses will be larger.

If the role set gets too large, enforcing policy may take marginally
longer.


Other Deployer Impact
---------------------

* Change takes effect immediately, but no implied roles will be
  defined by default.

* Without a configuration option change, no role
  inference will be performed.


Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    ayoung


Other contributors:
    None

Work Items
----------

All code changes must be in the Assignments back-end.
*  Add parent field to entities
*  Expand the create and edit implied role APIs
*  Add notifications
*  Account for hierarchy on listing role assignments

Dependencies
============

None

Documentation Impact
====================

Documentation of RBAC will need to cover hierarchies of roles.


References
==========

NIST RBAC
http://csrc.nist.gov/groups/SNS/rbac/
http://csrc.nist.gov/rbac/sandhu-ferraiolo-kuhn-00.pdf

Adding Attributes to Role-Based Access Control
http://csrc.nist.gov/groups/SNS/rbac/documents/kuhn-coyne-weil-10.pdf


ABAC and RBAC
http://csrc.nist.gov/groups/SNS/rbac/documents/coyne-weil-13.pdf

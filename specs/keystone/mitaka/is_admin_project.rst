..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Annotate Tokens for the admin project
=====================================

`bp example <https://blueprints.launchpad.net/keystone/+spec/is-admin-project>`_


Add an additional identifier to validation responses for tokens that are
scoped to the designated 'admin' project.


Problem Description
===================

Keystone's rbac model grants roles to users on specific projects, and
post-keystone redux, there are no longer "global" roles.

There are, however, circumstances where there is a need for a token that would
allow a user to undertake a broader set of operations than related to a single
project. Examples are:

* Operating on resources that are not project related
* Providing cloud admin capability for select users

Today there are two ways this can be achieved. The first is how all default
policy files are written - in that if a user has the role `admin` on any
project, they effectively can be admin on any other project. This is clearly
undesirable. In reality, these policy files should be checking the scope of
the project in the token to ensure that the user really does have the
appropriate role on the project being operated on. However, doing that would
again prevent either of the two examples of above being solved.

The second method is used by the keystone policy.v3cloudsample.json example,
where a specific domain is "blessed". If you have the role `admin` on this,
special domain, then you can act as a cloud admin or operate on resources that
are not project specific. The only problem with the approach in
policy.v3cloudsample.json is that it requires, at setup time, the ID of the
special admin domain to be pasted into one of the rules in the policy file:

"cloud_admin": "role:admin and domain_id:<insert admin domain_id here>",

The use of a special project or domain seems an appropriate way of providing
the functionality required, but it would be preferable not to have to modify
all the policy file of the services with a specific project/domain ID.


Proposed Change
===============

It is proposed that keystone support a configuration option to specify a
special admin project. If defined, any scoped token issued for that project
will have an additional identifier `is_admin_project` added to the token. This
identifier can then be checked by the policy rules in the policy files of the
services when evaluating access control policy for an API.

One immediate question that might be raised at this point is what if we want to
have a different admin for say nova, neutron, cinder etc. who could operate on
non-project related resources in those services. Does this mean we would need
to support a special project per service? The answer is No, and this kind of
support could be obtained by:

* Creating roles for admin of each service (e.g. `nova_admin`, `cinder_admin`)
  which would be checked for in each of the service policy files, e.g.:

"i_am_nova_admin": "is_admin_project and role:nova_admin"

* Writing the keystone policy rules for role assignment such that only those
  users with the actual role `admin` on the special project can assign other
  roles to that project (i.e. create nova, cinder, glance admins).

A further question is how this relates to the current keystone v3 sample that
uses a special admin domain. With the move to representing domains as projects
with the `is_domain` attribute in Mitaka, keystone can use this facility
to provide the same cloud admin support, e.g.

"cloud_admin": "role:admin and is_domain and is_admin_project"

Indeed, with this in place, we should be able to make the v3 policy sample the
default keystone policy file (although a separate spec will be produced for
that change).

Since we expect v2 tokens to be supported for a significant time, it is
proposed that `is_admin_project` will be supported for both v2 and v3 tokens.

Alternatives
------------

* Only issuing `admin` roles on tokens for admin projects. Fixes
  this problem but introduces far worse ones, as many APIs assume that
  users can get a token scoped to a project with the `admin` role.

* Make roles global.  Breaks many existing Policy checks that assume
  the roles to be scoped to existing projects.  Does not really fix anything.

* Templatize the policy file in the same way as the v3cloud sample, and use
  the deployment specific configuration management system to keep them in sync.
  This requires big changes through a lot of the tool chain.

Security Impact
---------------

This change has a positive security impact since it allows deployers to
significantly reduce one of the existing risk areas - where policy files might
grant admin rights to more users than was intended. However, to take advantage
of this mechanism requires a one-time modification of all policy files for
remote services.  As such, it will have no effect when initially implemented.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.  It will allow users with `admin` on the admin project to
continue to do a wide range of activities, and will only limit users
that have that role on other projects.

Performance Impact
------------------

Minimal, if at all.  Requires an additional check of the project_id
when creating a token.

Other Deployer Impact
---------------------

Will require two config values: `admin_domain_name` and
`admin_project_name` to allow the specification for the `admin` project. If
only `admin_domain_name` is specified, then the project acting as that
domain will be used.

Developer Impact
----------------

Should drive closing https://bugs.launchpad.net/keystone/+bug/968696
for all projects.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
ayoung <Adam Young> ayoung@redhat.com


Work Items
----------

* Implement changes in Keystone server token issuance

* Modify policy files in all other projects


Dependencies
============

None


Documentation Impact
====================

Documentation will have to indicate how to set the `admin_domain_name`
and `admin_project_name` to limit the scope of admin tokens.


References
==========

* https://bugs.launchpad.net/keystone/+bug/968696

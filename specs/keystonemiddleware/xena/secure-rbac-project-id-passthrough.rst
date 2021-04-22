..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
secure-rbac - X-Project-Id Pass-through
==========================================

`bug #1925684 <https://bugs.launchpad.net/keystone/+bug/1925684>`_

As was discussed during the Wallaby PTG [1], implementing secure-rbac policies
for system scoped credentials can be challenging for projects where some APIs
require a project ID.

This spec proposes a change to Keystone middleware to provide a project ID
to be used in the context of such APIs.  This should enable projects to
write and use system scoped policy rules with no code changes in most cases.

From an end user perspective, this will allow operators to easily manage
resources owned by any project.

Problem Description
===================

System scoped tokens issued by Keystone do not contain a Project ID.  This is
problematic for services where their API impementations assume that a UUID
will always be available.

Attempting to use system scoped tokens with such APIs typically results in
errors or unwanted behavior.  For example, some Barbican API calls return
5XX internal errors for operations that assume that there is a Project ID
in UUID format in the request context.

Similarly, Nova also returns 500 errors when requests are sent using tokens
that do not provide a Project ID. [2]

Proposed Change
===============

Currently, keystone middleware modifies the request after authentication
by adding context headers from the authenticated data.  For project scoped
tokens, it adds a X-Project-Id header (which is the same as HTTP_X_PROJECT_ID
from the WSGI point of view).  For system scoped tokens, this header
is not added, which results in a `None` value in many cases.

For security reasons, the middleware currently removes a number of headers
before authentication, including X-Project-Id.  This makes sense as we want to
prevent a malicious user from trying to inject unauthenticated context data
into the request.

The proposed solution will change the "sanitation" behavior of removing the
X-Project-Id every time to allow it to be passed through for requests made
with system scoped credentials:

* If present, the X-Project-Id header is cached
* Provided credentials are authenticated
* When the provided credentials are project-scoped the cached value is
  discarded, and the value from the authenticated data is used
* When the provided credentials are system-scoped the cached value is
  added to request in the X-Project-Id header.

Alternatives
------------

Two other alternatives were discussed:

#. Wrap operations on project-owned resources with explicit role assignments.

    Advantages:

    * Explicit audit trail via role assignments in keystone
    * Ensures that operations on project-owned resources must be done
      with a project-scoped token

    Disadvantages:

    * Very chatty and will increase the amount of time it takes for system
      users to do things with project-owned resources
    * Will not work for system-users cleaning up resources in a project that
      doesn't exist since you can't get a token scoped to a project that
      doesn't exist.  i.e. You can't clean up orphaned resources after a
      project has been deleted from Keystone.
    * Could result in a leftover role assignments for system users
    * Will not work for system readers since they don't have the ability to
      create role assignments in keystone (that's a writable operation)

#. Use inherited role assignments from domains.

    Advantages:

    * Less chatty than approach #1 because it doesn't require an explicit
      assignment
    * Ensures that operations on project-owned resources must be done with a
      project-scoped token

    Disadvantages:

    * Must be done for each system user on each domain, which are both dynamic
      resources (e.g., requires a process doc?)
    * Potential for privilege escalation if a system reader is given the admin
      or member role on a domain (what's responsible for this, the deployment
      tool? the user?)
    * Will not work for system readers since they don't have the ability to
      create role assignments in keystone (that's a writable operation)

Refer to the Xena notes [1] for details of their advantages/disadvantages.

One key difference between both of these alternatives and this spec is that
this spec does not require Keystone to issue a project token to be used by the
client.

Instead, this spec allows for enforcement of RBAC rules that target system
scope and system roles assigned to that user while still providing a Project ID
for context.

Security Impact
---------------

As mentioned, removing X-Project-Id as the middleware currently does is good
practice.  We should consider any adverse effects of allowing pass-through for
system scoped requests.

It's also important to be able to audit when system scoped requests are using
this pass-through.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Clients will need to be updated to be able to provide a Project-Id in a
request when using system scoped tokens. e.g.

    openstack secret list --os-project-id XXXXX-XXXX-XXXX-XXXX

Should add that project ID to the request headers before sending.

KSM should fail if the request has more than one Project ID.  The reasoning
for this is that currently only one project ID is added to the headers because
any X-Project-Id headers are removed from the request.  Allowing more than
may cause unintended side effects and/or errors.

Performance Impact
------------------

There should be no performance impact from this change as it only affects
system-scoped users (operators).

Other Deployer Impact
---------------------

None

Developer Impact
----------------

This should hopefully allow APIs that have built in assumptions about the
project ID in the context to work with system scoped requests without any
code changes.

Implementation
==============

Assignee(s)
-----------

* Douglas Mendizábal (redrobot)
* Lance Bragstad (lbragstad)

Work Items
----------

* Implement middlware changes
* Implement client changes
* Implement additional changes to keystoneauth to make sure it populates the
  same header (HTTP_X_PROJECT_ID) if context.system_scope and
  context.project_id are not None.
  This is important for service-to-service communication (e.g., nova using the
  user's context object to talk to neutron for a port binding when building a
  server for a specific project).

Dependencies
============

None

Documentation Impact
====================

Documentation will need to be updated to specify when Project ID values
are allowed to be passed through.

References
==========

[1] https://etherpad.opendev.org/p/policy-popup-xena-ptg
[2] https://bugs.launchpad.net/nova/+bug/1918945

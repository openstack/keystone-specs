..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Role Check From Keystonemiddleware
==================================

`bp Role Check from Keystonemiddleware
<https://blueprints.launchpad.net/keystone/+spec/role-check-from-middleware>`_

Role Based Access Control (RBAC) requires administration of both the
roles assigned to users and the rules that determine what role can
perform what action. To date, OpenStack has made role assignment
fairly easy to use, but modification of policy files has been manual,
decentralized, and inconsistent.

The goals:

 * Allow operator assignment of the roles to operations
 * Provide a means to report what role is required for an operation
 * Allow users to delegation subsets of their roles, potentially
   allowing them to delegate the ability to perform individual
   operations


Problem Description
===================

The authorization data associated with Keystone tokens contains a set
of roles that can be used to enforce access control. This is a
departure from the NIST definition of Role Based Access Control
in that the role names are only part of the overall role; they are
further scoped to the projects. A user assigned a role in one project
would not have access to a resource in another project. Thus, we call
this `Scoped RBAC`.

The default OpenStack deployments make very little use of RBAC.
The 'admin' role is the only role that is explicitly checked across
various OpenStack project policy files. The 'admin' role has been
unclearly scoped to both global and project scoped operations. While
the policy.json files are supposed to be configuration files, and
editable by the end deployers, the reality is that this is difficult,
and even discouraged in the official documentation.

There are numerous challenges to updating the current policy
files. Changing policy now requires redeploying configuration files
for each node in the service. Applying changes to a role requires
coordination between keystone and the service configuration. Certain
operations require other operations in order to be successful, so if
the policy fails on a downstream operation the whole operation
fails. This is too high a risk for most deployment.

Implementing a dynamic RBAC policy mechanism inside OpenStack has to
work within the restrictions of a distributed development model. Any
approach which requires changes to every project has little to no
chance of succeeding. Thus, RBAC enforcement needs to be encapsulated
with Keystone and Keystonemiddleware. However, the full policy check
as performed by policy.json and oslo-policy is embedded deep within
the code of each project, due primarily to the need to fetch a record
from a database to check attributes.

When looking at most of the policy files, they either check that the
user has the admin role, or that the user has any role on the
project. The matching logic in the policy.json rules are very specific
to each of the projects. There is little reason to modify this part
of the policy. In fact, doing so might break deployments upon update.

It would be possible to have an additional call to oslo-policy
from keystonemiddleware to check only the roles. However, that leaves
additional questions unsolved: how do we make the role checks easily
editable, but still distributed to all of the services?

The RBAC check does not require the same information required for the
scope check. While the scope check requires attributes off a resource
fetched from the database, the RBAC check can be performed entirely
off of information involved in the request. The basic data required
is:

  * The requested URL, to include the understanding of which service
    or endpoint the URL implements.
  * The Data returned from the Token

Proposed Change
===============

Overview
--------

Perform a role check in keystonemiddleware after the token validation
by using a set of rules that map from VERB + URL Patterns to a small
set of roles, and then expanding that to a full set of roles via the
role inference rules.

The RBAC check happens before keystonemiddleware passes control to
the service specific code. Leave the current oslo-policy based access
checks in place, using the existing policy.json files. This leads to a
separation of concerns: Middleware enforces the role check, source
code enforces the scope check.

The following changes are required to enable the RBAC check:

  * Create persisted entities in Keystone that contain

    - the service name

    - the HTTP Verb of the Request

    - the URL pattern

    - a minimum of a single required role

  * Create an API for upload and modification of these entities, to
    include bulk upload per service.

  * Deduce the values from the Documented APIs to Create instances via
    the above APIs.

  * Perform a Role check in keystonemiddleware after the token validation
    that uses the role to URL Pattern mappings to ensure that the user
    has the mapped role. Roles will be expanded via the role
    inference rules.

While some policy files currently make checks for specific roles,
most notably the 'admin' role, these apis are specifically
identitfied as requiring that role. Those checks will remain in place.

RBAC Check Flow
---------------

For an example, I am going to use the Nova call to modify a virtual
machine. A user would make an HTTP call such as

.. code-block:: bash

   curl -H "X-Auth-Token: adb5c708a55f" \
     -H "Content-type: application/json" \
     PUT https://nova1:8774/v2.1/2497f6/servers/83cbdc \
     -d @new_values.json

With the body of the request inside the @new_values.json file.


Middleware
~~~~~~~~~~

Inside the web server, the WSGI application runs through a set of
middleware classes until it reaches `keystonemiddleware.auth_token`.
The auth_token class reads the [keystone_authtoken] section of the provided
configuration file. A new key has been added: `service` which will be
used to select the appropriate set of api_roles for matching.


Here is an example
.. code-block:: ini

   [keystone_authtoken]
   username=nova
   project_name=service
   auth_type=password
   auth_url=http://192.0.2.3:35357
   password=7c48bc8f2668001d81582506f7c83d242f62502e
   service=compute

Fetch RBAC Data
~~~~~~~~~~~~~~~

After the token has been validated via a call to Keystone, the
middleware will fetch the RBAC specific data via python-keystoneclient
which calls the API. Due to caching needs, this result will be stored
in cache so that the response can also be loaded directly from its
JSON representation.

.. code-block:: bash

   GET https://hostname:port/v3/api_roles?service=compute

An example of a subset of the response data is shown below:

.. code-block:: json

   {
       "service": "compute",
       "api_roles": [{
           "verbs": ["POST"],
           "pattern": "/servers/{server_id}/action",
           "roles": ["Member", "admin"]
       }, {
           "verbs": ["POST"],
           "pattern": "/os-cells",
           "roles": ["admin"]
       }, {
           "verbs": ["GET", "PUT"],
           "pattern": "/v2.{subversion}/{tenant_id}/servers/{server_id}",
           "roles": ["Member", "admin"]
       }],
       "default": {
           "roles": ["Member", "admin"]
       }
   }


Role check
~~~~~~~~~~

For this example, we will specify that the expansion of role
inference rules in the token response is disabled. This will minimize
the token response data size as the number of defined roles increases.

keystonemiddleware.auth_token will use python-keystoneclient to make a remote
query against the keystone `api_role` API passing in the parameter
`service` to get the approprate set of rules.

By the time the code has passed to Keystonemiddleware, the complete
URL will have been processed by the WSGI pipeline, removing the
Hostname and port. The remainder of the URL may start
with the version information in the pattern /v[0-9.]*/.
In our example, this leaves: `/v2.1/2497f6/servers/83cbdc`.
The pattern matching will be run against this sub-url.

keystonemiddleware will iterate through the set of api_roles,
attempting a match against each one. The URL remainder above will
match the pattern

GET /v2.1/{tenant_id}/servers/{server_id}

Since the token response will have the role "Member" which matches the
set of roles: `roles=["Member", "admin"]` the validation will succeed.

If none of the URLs match, or if the auth-data does not
contain a role from the set specified by the pattern, validation
fails. The failure path will be similar to a failed token validation.

After the token and RBAC validation is completed successfuly, there is
no change to existing processing. There is no change to the set of
additional headers that middleware adds to the context. The WSGI
middleware pipeline continues, eventually calling into the Nova server
specific code. Inside this code, Nova will call the oslo-policy
library to enforce policy as specified by either the Nova annotations
or the overloads provided in the policy.json or policy.yaml files.

Object Schema
--------------
The new entity stored in the database would have the following layout.

api
~~~
ID: Autogenerated UUID
Service: Indexable String, matches the values from the service catalog
Pattern: Long String (>255 chars) that contains the patterns.
role_id: UUID index to the role table
verbs: Long String (>255 chars) storing the serialized json array of the set
of verbs.

api_role
~~~~~~~~
API_ID: Indexable String foreign key to the api table
role_id: Indexable String foreign key to the role table

The service, pattern, and verb fields may be null. Catch all rules
will be stored in the database using the same schema.  null values
will be treated as wildcard.

Thus the default above maps to a row in the table with

.. code-block:: json

   {
       "id": "bcd321",
       "service": "image",
       "pattern": "None",
       "verb": "None"
   }

As well as an api_role entity with an api_id of `bcd321` and a
role_id that corresponds to the "Member" role.

If there are no `api_role` entities definied for an `api` entity,
the result will have a special value of `None` for the roles will
indicate that the no Role is required, and that the entire token role
check can be skipped.  This will allow operations that do not require
a token, or that are allowed to work with an unscoped token, to
procede. This example shows how to allow version discovery to procede.

.. code-block:: json

    [{
        "service": "identity",
        "pattern": "/v",
        "verb": "GET",
        "role": "None"
    },
    {
        "service": "identity",
        "pattern": "/v3",
        "verb": "GET",
        "role": "None"
    }]


A global catch all rule can be defined for requests for services that
have not been yet defined.

.. code-block:: json

    {
        "service": "None",
        "pattern": "None",
        "verb": "None",
        "role": "None"
    }

This rule will only be returned only for queries that have
no `api_role` entries defined.

The Keystone bootstrap process will define this initial api_role.

Bulk Upload and Query of API Roles
----------------------------------

Initialization of a system requires a set of rules for each of the
services. These rules should be maintained by the core team for each
service, and modified by the end deployer.


A sample of a subset of
the rules for glance could look like this:

.. code-block:: json

   {
       "service": "image",
       "api_roles": [{
           "pattern": "/v2/images",
           "verbs": ["POST"],
           "role": "member"
       }, {
           "pattern": "/v2/images/{image_id}",
           "verbs": ["GET", "PATCH", "DELETE"],
           "role": "member"
       }, {
           "pattern": "/v2/metadefs/namespaces/{namespace_name}/objects",
           "verbs": ["POST"],
           "roles": "admin"
       }, {
           "pattern": "/v2/metadefs/namespaces/{namespace_name}/objects",
           "verbs": ["GET"],
           "roles": "member"
       }, {
           "pattern": "/v2/images/{image_id}/deactivate",
           "verbs": ["POST"],
           "role": "member"
       }, {
           "pattern": "/v2/images/{image_id}/reactivate",
           "verbs": ["POST"],
           "role": "member"
       }],
       "default": {
           "roles": ["member", "admin"]
       }
   }


Offline Role query
~~~~~~~~~~~~~~~~~~

If a user wishes to be able to deduce what role they need to perform
an operation, they can fetch the `api_roles` from Keystone, find
the pattern that matches, and what role it requires. For example, if
a user wanted to create a trust for a service that was going to have
to check a block device, they could take the URL:

.. code-block:: bash

   https://cinder:8776/v1/f0123/volumes/a0321

Which would match the pattern below:

.. code-block:: json

  {
      "service": "storage",
      "api_roles": [{
          "pattern": "/v1/{tenant_id}/volumes/{volume_id}",
          "verbs": ["GET"],
          "role": "auditor"
      }]
  }


and determine they need to delegate the `auditor` role. Assuming the role
inference rule that states `Member` implies `auditor`, a user with the
`Member` role can then create a trust with the implied `auditor` rule
for the remote service.

For a Web UI like Horizon, this method could be used to customize the
User interface, to determine if a class of resources should be shown,
and whether or not they are editable, based on the roles of the user
and the APIs needed to populate that page.


Alternatives
------------

Many of the earlier proposals have attempted to work with the existing
policy structure. Several proof-of-concepts have been written that
dynamically fetch the oslo-policy, or remotely execute a comparable
check.

The main reason for not pursuing this approach is that it is very hard
to abstract it while continuing to provide the full set of data
required. For example, project Moon was able to make
a check work based on the URL only, it did not actually have the
Server data from the database at middleware time. Also, the amount of
administration, especially the definition of attributes, meant that
the domain structure from Nova was duplicated in the Keystone
Database.

The current approach to scoping policy can be described as "all
resources of the same type withing a project have the same access
control." Several projects, most notably around credentials in
Barbican and Keystone, have attempted to enforce more fine grained
policy than the current approach, specifically, based on the user that
created the object. However this has been shown to be problematic at
cloud scale. Any delegations created that attempt to use those
objects must now use impersonation, which is dangerous. To clean up
these resources, should that user not be present is to escalate it to
an administrator.

The RBAC approach described here does not prescribe
such an approach, it just takes a more pragmatic and scalable approach
first. This approach better matches the OpenStack design.

Other specs that have addressed this are listed in references.




Security Impact
---------------

The intention is that this will improve security. If checks are
improperly implemented, however, it could lead to weak role checks.

One potential benefit will be the ability for users to create
delegations with only the subset of roles required for the operation.
For example, if a user only wants a watchdog program to kill a VM if
it misbehaves, the administrator could create a role called
`compute_delete_server` specific to the API `DELETE
/v2.1/{tenant_id}/servers/{server_id}` as well as a role inference
rule.


Here is an example
.. code-block:: ini

  member: compute_delete_server
  compute_delete_server: DELETE /v2.1/{tenant_id}/servers/{server_id}

The user could then create a trust with only the role
compute_delete_server specified.

Since the Nova policy file only checks that the project ID matches, and
does not do any explicit role check, the nova policy file would remain
unchanged.


Notifications Impact
--------------------

Notifications for changes to API api_roles will be comparable to the
notifications for the Roles API.

Notifications due to failed token validations now will also include
those that are from failed RBAC checks.


Other End User Impact
---------------------

APIs should behave just like they have before.

If the number of implied roles increases significantly, it will be
impractical to continue to expand them in the token validation
bodies, as this will greatly increase the size of the response.
Since the expansion of implied roles will happen in the response for
list URL Api_Roles, people with this issue should disable expanding
toles in the token.

Example:

Assuming an implied role chain like this: `r1->r2->r3->r4->r5->r6->r7`

And an URL pattern rule like this:

.. code-block:: json

   {
       "pattern": "/v2/images/{image_id}/reactivate",
       "verbs": ["POST"],
       "role": "r7"
   }

The implementation will have an api_role response that
looks like this:

.. code-block:: json

   {
       "pattern": "/v2/images/{image_id}/reactivate",
       "verbs": ["POST"],
       "roles": ["r1", "r2", "r3", "r4", "r5", "r6", "r7"]
   }

while the token validation response would only have:

.. code-block:: json

   {
       "roles" :[{"name":"r1"}]
   }


Performance Impact
------------------

The overall performance impact here is hard to judge. Here are some
issues that have been discussed thus far.

There would be a small, but non-zero impact in the remote service due
to the need to fetch and cache the RBAC data. Since the API matching
rules fetched from the Keystone server will likely be cached in
the remote server, there should be minimal impact on the Keystone side
due to database lookups.

Evaluating the rules would require a linear match, much the same way
that a router does in Keystone. The longer the set of APIs, the
longer it will take to match. More complex matching schemes based on
the API roles rules can potentially optimize this if it proves to be a
problem.

One positive impact is that, for tokens without valid roles, code that
would have, in previous cases, called into the database layer of the
services will no longer have to do so. The RBAC check will go to the
Keystone server prior to the object being fetched from the database.


Other Deployer Impact
---------------------

Deployers will now be able to deploy their own policies for just the
RBAC stage. Since this requires configuration changes to activate, no
change in behavior will happen until the changes are made.

Once the code changes are in place, the deployer will have to load the
rules to the Keystone server before relying on this mechanism. They
can either edit the rules before uploading them in bulk, or can use
the patch verb to upload modified URL-Pattern to role mappings for a
subset of the URLs.

While the API allows assigning multiple roles per API, the preferred
mechanism for managing what the required roles for an operation is to
define `implied-roles` that map from Admin or Member to an operation
specific role. These changes can be made without modifying individual
role assignments.

As an example, assume a site wants to implement a specific role for
read-only operations, and to start, wants to implement it for the
glance image GET operation. Assuming they started with the rule
above for the `image` service and `pattern` of
`/v2/images/{image_id}`, which is initialized to the member role the
deployer would do the following:

1. Create a new role with the name `reader`
2. Create a role inference rule that `member` implied `reader`
3. Change the api_role so that instead of requiring the `member`
   role it requires the `reader` role.

Retrieving the api_roles would result in the following entries.

.. code-block:: json

   [{
        "pattern": "/v2/images/{image_id}",
        "verbs": ["patch", "delete"],
        "role": "member"
    },
    {
        "pattern": "/v2/images/{image_id}",
        "verbs": ["get"],
        "role": "reader"
   }]


Developer Impact
----------------

The first pass of generating the new RBAC rules can be done using the
API documentation, as that lists the calls in the expected format.
This will get the majority of the APIS. Not all of the APIs are
documented this way yet.

These documents should be managed by the individual service git repos.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  Adam Young ayoung ayoung@redhat.com

Other contributors:
  * Jamie Lennox jamielennox jamielennox@gmail.com
  * Alexander Makarov amakarov amakarov@mirantis.com
  * Henry Nash henrynash henryn@linux.vnet.ibm.com
  * Ruan He ruan.he@orange.com


Work Items
----------

  * API for administration of API api_roles
  * Logic to match the current URL to the pattern and perform the
    role check implemented in python-keystoneclient
  * Composition of the default rules.
  * Extensions to the Token Validation API to allow for new parameters
  * Modification of keystonemiddleware.auth_token to fetch the
    role_apis and perform the RBAC check.

Dependencies
============

* No new dependencies are introduced by this change.


Documentation Impact
====================

  *  new APIs
  *  initialization and management of the role_api values
  *  upgrade procedures
  *  creating trusts, oauth, and role assignments based on least
     privilege


References
==========

Supporting Documents
--------------------

  * `Project Moon Proposal <https://wiki.opnfv.org/display/moon/Moon+Projet+Proposal>`_.
  * `NIST RBAC <http://csrc.nist.gov/groups/SNS/rbac/>`_.
  * `XACML <https://en.wikipedia.org/wiki/XACML>`_.
  * `RBAC Policy Update in Tripleo <https://adam.younglogic.com/2016/08/rbac-policy-update-tripleo/>`_.
  * `Implied Roles
    <https://specs.openstack.org/openstack/keystone-specs/specs/keystone/mitaka/implied-roles.html>`_.
  * `Domain specific roles <https://specs.openstack.org/openstack/keystone-specs/specs/keystone/mitaka/domain-specific-roles.html>`_.
  * `Dynamic policy in Keystone <https://adam.younglogic.com/2014/11/dynamic-policy-in-keystone/>`_.


Related Specs
-------------

  * `Complete RBAC in Keystone <https://review.openstack.org/#/c/325326/>`_.



Abandoned related Specs
-----------------------

These specification have all attempted to tackle some subset of the
same issues.

  * `Reservations  <https://review.openstack.org/#/c/330329/>`_.
  * `Policy Merge <https://review.openstack.org/#/c/295049/>`_.
  * `Policy Rules Managed from a Database <https://review.openstack.org/#/c/133814/>`_.
  * `Dynamic RBAC Policy <https://review.openstack.org/#/c/279379/>`_.
  * `Identify policy by hash <https://review.openstack.org/#/c/297897/>`_.
  * `Support RBAC with LDAP in oslo.policy <https://review.openstack.org/#/c/259418/>`_.
  * `Fetch Policy by Tag <https://review.openstack.org/#/c/298788/>`_.
  * `Policy rule name spacing via catalog <https://review.openstack.org/#/c/237743/>`_.
  * `Alternative policy enforcement <https://review.openstack.org/#/c/323791/>`_.
  * `Policy Mapping API <https://review.openstack.org/#/c/185126/>`_.
  * `API spec for managing Attribute hierarchies in the Policy
    database <https://review.openstack.org/#/c/184926/>`_.
  * `Basic API spec for managing Policy rules in a database
    <https://review.openstack.org/#/c/184903/>`_.
  * `API spec for searching on the Policy database
    <https://review.openstack.org/#/c/186093/>`_.

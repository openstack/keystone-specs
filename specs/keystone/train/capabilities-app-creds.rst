..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Add Fine Grained Restrictions to Application Credentials
========================================================

`bp <https://blueprints.launchpad.net/keystone/+spec/whitelist-extension-for-app-creds>`_

Currently Keystone application credentials are mostly unrestricted.
Restrictions can only be imposed on creation of follow-up application
credentials and trusts. Other than that they allow unfettered use of the roles
being delegated in the project the application credential is created for. This
renders application credentials questionable anywhere a least-privilege
delegation is desired. Technically it would be possible to store a white list
style list of access rules for an application credential which other OpenStack
services would then enforce. This spec outlines an approach for storing and
handling such restrictions.

Problem Description
===================

This section uses the predecessor of application credentials, Keystone trusts,
to outline a few example where unrestricted role delegation is a problem. The
same problem applies to application credentials since both delegate a subset of
users' roles. Only the authentication method is different (a secret for
application credentials, a trustee user's username/password for trusts).

Keystone trusts, have been around for quite a while now (they were introduced
in the Grizzly release). Right now, trusts are used by the following projects
among others (list may not be complete):

* Heat: operations on behalf of the user at times when a token may have expired
  already.

* Magnum: access to Magnum's certificate signing API and other OpenStack APIs
  from inside a cluster's instances where the container orchestration engine
  requires it (e.g. Glance as backend for docker-registry or cinder as backend
  for docker-volume)

Other projects (neutron-lbaas, Barbican) hesitate to employ trusts and
application credentials since they are an all-or-nothing approach: they grant
full access to all OpenStack APIs in the scope (roles in a project) they are
created for. In order to provide least-privilege access, these services
implement ACLs of their own (Barbican, Swift) or rely on other services' ACLs
to grant limited access to resources (neutron-lbaas uses Barbican's ACLs to
grant its service user access to secret containers holding SSL keys). Monasca
suffers from slightly different problems: it uses Keystone to authenticate
metric submission which requires Keystone credentials or an application
credential. This can potentially be abused for out-of-band access to other
OpenStack APIs from inside any VMs running a Monasca agent.

In summary, there is a real need for fine-grained delegation of access. The
implementation of Keystone application credentials as it exists right now
cannot serve this need, though: an application credential can only be used to
grant full access within the application credential's project/roles scope, but
it cannot be used to give access limited to just one particular resource or
access that only allows the creation of specific new resources, but not the
modification/deletion of existing resources.

With fine-grained restrictions in place, application credentials can be used to
safely grant the least privilege required in the scenarios described above.
This is not possible with the current role based access control solution some
services use, where a special-purpose role such as Heat`s `heat_stack_user`
role is merely explicitly blacklisted for all operations other than the
specific one it is intended for. For this blacklisting usually only extends to
the service owning this particular role and putting these restrictions in place
(other services usually do not know this role is supposed to have blacklist
entries for any and all of _their_ operations and thus allow unrestricted
access).

Proposed Change
===============

The approach to implementing fine-grained permissions for application
credentials is two-pronged. Permission data is stored in Keystone and enforced
by keystonemiddleware as follows:

1) Alongside an application credential, a list of access rules with zero or
   more access rules can be stored. An entry in this list consists of:

   (a) A URL path (e.g. `/v2.1/servers`, `/v2.1/servers/*` or
       `/v2.1/servers/{server_id}`). This URL path must be explicitly permitted
       according to an operator-configured list of access rules (see `Access Rules
       Config`_ below).
   (b) A request method (e.g. `GET`)
   (c) A service type (ideally matching the `published Service Types Authority`_)
       from the Keystone service catalog.

   This list is a whitelist, i.e. any request not explicitly allowed by an
   access rule is rejected. Keystone itself does not validate the content of
   access rules because that would require domain knowledge of each service in
   the catalog.

   The access rules are stored in a separate database table and linked to the
   application credential so that old rules can be re-used with new application
   credentials.

2) `keystonemiddleware` on the service's side receives the access rule list
   during token validation. It then checks

   (a) The service type (e.g.  `compute`)
   (b) The URL path (e.g. `/v2.1/servers/*` or `/v2.1/servers/{server_id}`
       or `/v2.1/servers/b2088298-50e5-4c81-8a50-66bfd1d8943b`)
   (c) The request method (e.g. `GET`)

   Against every entry in the access rule list retrieved from the token. If an
   access rule matches the request, checking stops and the request is handed over
   to `oslo.policy` for the regular role based checking. If no access rules
   match, the request is rejected right away.

   There are three special cases to access rule list processing:

   (a) If no list is provided (i.e. if the `access_rules` attribute is
       `None`), no access rule checking is performed and the request is passed
       to `oslo.policy` right away.
   (b) If an empty list is provided (i.e. `[]`), all requests are rejected
       (even if the request would otherwise pass the test in (c).
   (c) If there is a valid service token in the request, `keystonemiddleware`
       passes the request to `oslo.policy` right away, though a future iteration
       of this feature will enable a toggle to control this behavior.

.. _published Service Types Authority: https://service-types.openstack.org/

Preventing Regressions
----------------------

If a Keystone API which supports this feature encounters a `keystonemiddleware`
version (or 3rd party software authenticating against Keystone) that dates to
before implementation of this feature there is potential for regression: while
Keystone would provide the access rule list upon token validation, the other
side would simply ignore it - giving the requests all the permissions granted
by the delegated roles. This can be prevented by treating application
credentials with access rules (i.e. a `access_rules` attribute that is not
`None`) as follows):

1) When requesting token validation, `keystonemiddleware` (or any 3rd party
   application that supports access rule enforcement) sets an
   `Openstack-Identity-Access-Rules` header with a version string as its value.
   Token validation for an application credential with a access rule list will
   only succeed if this header is present. The version string will allow us to
   safely extend this feature by invalidating tokens using the extended version
   in situations where `keystonemiddleware` only supports an older version
   of this feature.

2) If there is no `Openstack-Identity-Access-Rules` header in the token
   validation request, token validation fails.

This way we ensure that nobody erroneously assumes access rules are being
enforced in environments where outdated `keystonemiddleware` (or its equivalent
in 3rd party software) cannot enforce them because it is not aware of them. For
any application credentials that do not have access rules, validation proceeds
as it would have before the introduction of access rules (regardless of whether
there is an `Openstack-Identity-Access-Rules` or not).

API Examples
------------

An example creation request for an application credential looks as follows::

    POST /v3/users/{user_id}/application_credentials

.. code-block:: json

    {
        "application_credential": {
            "name": "allow-metrics-logs",
            "description": "Allow submitting metrics and logs to Monasca",
            "access_rules": [
                {
                    "path": "/v2.0/metrics",
                    "method": "POST"
                },
                {
                    "path": "/v3.0/logs",
                    "method": "POST"
                }
            ]
        }
    }

With this, two new access rules will be created under the user's ID. They can be
queried like this:

Request::

    GET /v3/users/{user_id}/access_rules

Response:

.. code-block:: json

    {
        "access_rules": [
            {
                "id": "180e86bc",
                "path": "/v2.0/metrics",
                "method": "POST"
            },
            {
                "id": "03e13d17",
                "path": "/v3.0/logs",
                "method": "POST"
            }
        ]
    }

If desired, they could then be re-used for another application credential by
providing the ID::

    POST /v3/users/{user_id}/application_credentials

.. code-block:: json

    {
        "application_credential": {
            "name": "allow-just-metrics",
            "description": "Allow submitting only metrics to Monasca",
            "access_rules": [
                {
                    "id": "180e86bc"
                }
            ]
        }
    }



Alternatives
------------

1) One alternative to this exists already: internal ACL implementations by
   various OpenStack services. This situation is undesirable for several
   reasons, some of which are:

     (a) Auditability: authorization information is stored in multiple
                       locations, all of which need to be checked to find out
                       who is authorized to perform what operation. From an
                       auditability perspective it would be preferable to have
                       a central source of truth.

     (b) Maintenance: when there are multiple independent implementations a lot
                      of code is duplicated and bugs may be duplicated as well
                      as new projects implement their own ACL system.

     (c) Consistency: with multiple sources of truth, an individual service's
                      ACLs may well end up overriding a cloud-wide policy
                      permitting or denying an operation.

2) `391624 <https://review.openstack.org/#/c/391624/>`_ proposes a
   superficially similar role check in `keystonemiddleware`. There are several
   key differences, though:

     (a) Application credential access rules do not require a `Cambrian
         explosion <https://en.wikipedia.org/wiki/Cambrian_explosion>`_ of
         fine-grained roles (one for every API operation of every OpenStack
         service) that must be managed by an administrator.
     (b) Application credential access rules does not require any changes to
         existing policy enforcement. Instead, they add an additional check
         that takes place before policy enforcement even comes into play and
         rejects requests early. Not being entangled with policy enforcement
         gives us the freedom to start out with a very basic implementation and
         add features as required later (as opposed to having to be feature
         complete immediately).
     (c) The role check in `keystonemiddleware` targets administrators who want
         to create role profiles for their users, such as "give this user
         read-only access to any services' resources but without letting them
         create new ones". Application credential access rules on the other
         hand, target OpenStack services and third party applications that only
         need access to a select handful of operations such as "submit SSL
         certificates to the Magnum API for signing".
     (d) Application credential access rules do not require keystone to be the
         guardian of access control rules, since all the information needed to
         validate access is contained in the token.
     (e) Unlike a policy based check, an access rule based check will also work
         for services that do not use `oslo.policy` such as Swift.

3) One implementation detail from the previous section was discussed at length
   at the Rocky PTG: one could have chosen to match for `oslo.policy` targets
   rather than URL paths in the access rules, which would have been easier in
   some ways. In the end we opted for url paths for the following reasons:

     (a) This is user facing and unlike API paths, policy targets are not
         easily discoverable by the user since there is no documentation on
         them. Moreover, policy targets are not as formalized as APIs and may
         easily change over time, thus breaking existing access rules.

     (b) URL paths can be rejected in keystonemiddleware, without involving
         `oslo.policy`, leading to a faster failure for unauthorized requests.

Future Considerations
---------------------

Chained API Calls
~~~~~~~~~~~~~~~~~

A future iteration of this feature may create a toggle to control whether a
service can use one of these tokens to make background requests on behalf of
the user, for example to allow the compute service to make requests to the
block storage service even though the block storage API wasn't explicitly
whitelisted in the application credential access rules. For the time being,
chained service requests like this will leverage service tokens to ensure
that subsequent requests made on behalf of a user will be completed as normal,
and will rely on operator-configured policies to prevent abuse.

Access Rules Config
~~~~~~~~~~~~~~~~~~~

A future iteration of this feature may enable a way for operators to restrict
the allowed access rules that a user may configure by creating a global
whitelist of access rules against which users' access rules are validated prior
to the creation of the application credential. The value of this would be to assist
users in creating valid access rules by validating them against known working
rules. It would also give the operator more control of the overall access
control configuration. However, for the time being, this feature is infeasible
because we lack discoverability of APIs and it is impossible to create a
complete list of valid access rules for all services across OpenStack and
external to OpenStack. Since providing a complete list is infeasible, leaving it
up to the operator to curate their own list causes a poor operating experience
for the operator and the list would be susceptible to mistakes, which in turn
would cause an extremely poor user experience for the end user.

When this feature becomes feasible, another possibility is to allow operators to
configure a role ID for each access rule to indicate that the user needs to
provide that role in the application credential in order for the call to
proceed. This allows for greater alignment between policy rules and access
rules.

Limitations
-----------

This proposal does not restrict the body of requests in any sort of way.

Security Impact
---------------

This change tightens security by providing a means to restrict the permissions
granted by application credentials. That being said, its implementation does
have various security critical aspects:

* This change adds additional information to the token data retrieved by
  keystonemiddleware upon token validation.

* URLs in access rules are user-supplied strings. Care must be taken to
  guard against format string attacks in these if anything beyond character by
  character comparison takes place.

* It might be a good idea to limit the length/number of access rules per
  API credential to prevent denial of service against the Keystone database (by
  filling it with bogus rules) or the Keystone API (via large validation
  payloads). Another reason to introduce such a limit is the possibility to
  slow down a service by creating application credentials with a large number
  of non-matching access rules, which can be used to slow down a particular
  service.

* This change is unlikely to allow privilege escalation since it only adds
  additional failing criteria to token validation and policy enforcement. These
  failing criteria need to be carefully tested for false positives, though.

Notifications Impact
--------------------

No new notifications will be added from this API.

Other End User Impact
---------------------

Since this changes adds extra information to application credentials, both
python-keystoneclient and python-openstackclient need to be extended to handle
that extra information.

Performance Impact
------------------

The performance impact upon application credential creation is probably
neglible, since all that happens is that a small amount of data is stored along
with the application credential.

That small amount of data may not be so small during the token validation,
though, resulting in multiple/more packets being sent in response to a
validation request, causing congestion and/or increasing latency. This can be
mitigated by limiting the number of access rules allowed per application
credential.

Developer Impact
----------------

This change provides developers across all OpenStack services with a means to
create application credentials with fine-grained permissions, allowing them to
delegate access to a user's roles according to the principle of least
privilege.

As far as the application credentials API is concerned, it will be fully
backwards compatible, since specifying access rules when creating an
application credential is optional: if none are specified, the `access_rules`
attribute will be `None`, leading to no access rule checks being performed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * Colleen Murphy <colleen@gazlene.net> cmurphy

Other contributors:

  * Adam Young <ayoung@redhat.com> ayoung

  * Johannes Grassler <jgr-launchpad@btw23.de> jgr-launchpad

Work Items
----------

1. Extend the application credential API and database schema in Keystone to
   allow for receiving and storing access rule lists.

2. Implement handling for access rules in python-keystoneclient and
   python-openstackclient.

3. Extend the Keystone token validation API to access rule lists upon
   upon token validation.

4. Implement the endpoint list check in keystonemiddleware.

Dependencies
============

None

Documentation Impact
====================

* The access rule related settings for application credentials need to be
  documented in the release notes and the admin guide.

* Documentation on access rules needs to be added to the *Application
  Credentials* section of the Keystone user documentation.

References
==========

* Etherpad with original proposal from the Barcelona 2016 summit:
  https://etherpad.openstack.org/p/ocata-keystone-authorization

* Etherpad with refined proposal from the Rocky PTG 2018:
  https://etherpad.openstack.org/p/application-credentials-rocky-ptg

* Spec for securing Monasca metric submission from inside VMs
  https://review.openstack.org/#/c/507110/ (would be greatly simplified by
  having access rules in application credentials)

* Documentation on Barbican ACLs:
  http://developer.openstack.org/api-guide/key-manager/acls.html

* Documentation on Swift ACLs:
  https://www.swiftstack.com/docs/cookbooks/swift_usage/container_acl.html

* Generating a list of URL patterns for OpenStack services
  http://adam.younglogic.com/2018/03/generating-url-patterns/

* Related concept for Istio:
  https://istio.io/docs/reference/config/authorization/istio.rbac.v1alpha1/#AccessRule

* Updated design discussion:
  http://lists.openstack.org/pipermail/openstack-discuss/2019-February/003031.html

* Notes from Train Forum session:
  https://etherpad.openstack.org/p/DEN-keystone-forum-sessions-app-creds

* Notes from Train PTG session:
  https://etherpad.openstack.org/p/keystone-train-ptg-application-credentials

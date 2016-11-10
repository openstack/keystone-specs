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
style list of capabilities for an application credential which other OpenStack
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
          from inside a cluster's instances where the container orchestration
          engine requires it (e.g. Glance as backend for docker-registry or
          cinder as backend for docker-volume)

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

1) Alongside an application credential, a capability list with zero or more
   capabilities can be stored. An entry in this list consists of:

     (a) A URL path (e.g. `/v2.1/servers` or `/v2.1/servers/{server_id}`).
         This URL path must be permissible according to a URL path template
         which must exist in the table of URL path templates (see `Permissible
         Path Templates`_ below).
     (b) A dictionary whose keys need to exactly match the placeholders in the
         URL path. Both extraneous and missing keys for one or more
         capabilities will cause application credential creation to fail.
     (c) A request type (e.g. `GET`)
     (d) A service UUID from the Keystone service catalog. This UUID is not
         user provided. Instead, it is filled in from the URL template this
         capability is validated against.

   This list is a whitelist, e.g. any request not explicitly allowed by a
   capability is rejected. Keystone itself does not validate the content of
   capabilities because that would require domain knowledge of each service on
   Keystone's path. Every capability must reference an row in the table
   described in the `Permissible Path Templates`_ section below. If one or more
   capabilities entries fail this test, API Credential creation will fail.

2) A boolean `allow_chained` attribute of the application credential (`False`
   by default) controls whether chained API calls, i.e. follow-up calls issued
   by a service as a result of an API call permitted by a capability.  This may
   only be set to `True` if all capabilities listed in the template were
   validated against an URL template with its own `allow_chained` attribute set
   to `True`.

3) `keystonemiddleware` on the service's side receives the capability list
   during token validation. It then performs templating on all entries and
   checks

     (a) The service's own service ID (e.g.
         `ae8a69ae-3bc2-4189-88be-c0b9ea6ef06f`)
     (b) The URL path (e.g. `/v2.1/servers/{*}` or
         `/v2.1/servers/b2088298-50e5-4c81-8a50-66bfd1d8943b`)
     (c) The request's type (e.g. `GET`)

   Against every entry in the templated URL list computed from the capability
   information in the token (See `URL Path Templating`_ for how it is
   computed). If a capability matches the request, checking stops and the
   request is handed over to `oslo.policy` for the regular role based checking.
   If no capabilities match, the request is rejected right away.

   There are three special cases to capability list processing:

     (a) If no list is provided (i.e. if the `capabilities` attribute is
         `None`), no capability checking is performed and the request is passed
         to `oslo.policy` right away.
     (b) If an empty list is provided (i.e. `[]`), all requests are rejected
         (even if the request would otherwise pass the test in (c).
     (c) If the application credential's `allow_chained` attribute is `True`
         and there is a valid service token in the request,
         `keystonemiddleware` passes the request to `oslo.policy` right away.

Permissible Path Templates
--------------------------

Every capability must be validated against a URL Path Template referenced by
UUID upon application credential upon creation. This section describes how an
operator defines such URL path templates and how they are used by Keystone.

The permissible URL path templates are operator configured through the Keystone
API and stored in a dedicated table in the Keystone database. Keystone will
document a curated list of URL templates for those APIs where such a thing can
be generated automatically. The operator can then use this list as-is in the
simplest case, or modify it for their local setup as they chose. For every URL
template the following information is stored:

1) A service UUID that matches one of the services in the Keystone catalog.
   This is copied to the capability verbatim. The service UUID is validated
   upon URL template creation: it must match an existing service's UUID. This
   UUID should not have a foreign key constraint so as not to create
   dependencies from the catalog on URL templates or the capabilities validated
   against them. If a service is deleted later, and a non-existent UUID is thus
   being referenced, keystonemiddleware will reject any capabilities
   referencing it since there is no service whose service UUID will match it at
   that point.
2) A UUID that serves as a unique resource identifier. This is used to
   reference the path template to use for evaluation when creating a
   capability. This reference is only used for validation upon application
   credential creation and not recorded as part of the application credential.
3) A URL template string, such as `/v2.1/servers/{server_id}`. The combination
   of this string and the service ID from (1) must be unique. It is anchored at
   the beginning of a path, i.e. capabilities' path attributes must fully match
   this pattern and may not be preceded or followed by extra characters. The
   template string may contain the following special wildcard templates:

   * `{*}`: allows arbitrary strings (excluding the `/` character) in
            capability enforcement.
   * `{**}`: allows arbitrary strings (including the `/` character) in
             capability enforcement.

   A user using a URL template containing wild cards for validating one of
   their capabilities may substitute the wild card by any string fulfilling the
   constraint imposed by the wild card. This allows the operator to be
   permissive in their URL templates (to the point of only having one "{**}"
   pattern in the most extreme case) and the user to be more restrictive than a
   wild card template in their capabilities.
4) A boolean `allow_chained` attribute (`False` by default). If this is `True`
   for all URL templates referenced when creating an application credential,
   that application credential's own `allow_chained` attribute may be set to
   `True`.
5) A list of template keys to be provided by the user (henceforth referred to
   as "user template keys").
6) A list of template keys to be provided from token context. (henceforth
   referred to as "context template keys"). The following are available:

     * `domain_id` UUID of the domain the Application Credential is scoped to
       (where applicable)
     * `project_id` UUID of the project the Application Credential is scoped to
       (where applicable)
     * `user_id` UUID of the user who created the Application Credential

Between (4) and (5) all template keys in the URL template string must be
covered. If this condition is not met, creation of the path template fails.

URL Path Templating
-------------------

`keystonemiddleware` receives the capability list information upon token
validation. It then processes each capability as follows:

1) All placeholders from the user template keys list are replaced by the
   corresponding values in the user provided dictionary of values in the
   capability.
2) All placeholders from the context template keys list are replaced by the
   corresponding values from token context.
3) Wild card placeholders (`{*}`) are left in place. These will be used during
   capability enforcement to match any string in the respective path component.

Preventing Regressions
----------------------

If a Keystone API which supports this feature encounters a `keystonemiddleware`
version (or 3rd party software authenticating against Keystone) that dates to
before implementation of this feature there is potential for regression: while
Keystone would provide the capability list upon token validation, the other
side would simply ignore it - giving the requests all the permissions granted
by the delegated roles. This can be prevented by treating application
credentials with capabilities (i.e. a `capabilities` attribute that is not
`None`) as follows):

1) When requesting token validation, `keystonemiddleware` (or any 3rd party
   application that supports capability enforcement) sets an
   `Openstack-Identity-Capabilities` header with a version string as its value.
   Token validation for an application credential with a capability list will
   only succeed if this header is present. The version string will allow us to
   safely extend this feature by invalidating tokens using the extended version
   in situations where `keystonemiddleware` only supports an older version
   of this feature.

2) If there is no `Openstack-Identity-Capabilities` header in the token
   validation request, token validation fails.

This way we ensure that nobody erroneously assumes capabilities are being
enforced in environments where outdated `keystonemiddleware` (or its equivalent
in 3rd party software) cannot enforce them because it is not aware of them. For
any application credentials that do not have capabilities, validation proceeds
as it would have before the introduction of capabilities (regardless of whether
there is an `Openstack-Identity-Capabilities` or not).

Discoverability for URL Path Templates
--------------------------------------

Any user with a valid auth token can list the operator maintained URL path
templates through the Keystone API. This allows them to discover the URL path
templates they can use for creating capability enabled application credentials.

URL Templates and Roles
-----------------------

URL path templates will have an optional ROLE_ID value. If this value is set,
it indicates the role that the user needs to provide in the application
credential in order for the call to proceed. In addition, if the role_id value
is set, the user will only be able to use the URL value in a capability if the
user has that role assigned, either directly, or as a result of an implied
role.

Chained API Calls
-----------------

One thing the capabilities make rather tough is chained API calls: if an API
call is permitted by a capability, but the service uses the same capability
restricted token to call other services' APIs, these will fail. While it would
be possible to circumvent this problem with additional capabilities to cover
the chained calls, that would be very poor ergonomics, especially for
operations with a large amount of chained API calls such as creating a Heat
stack.

To make it easier on users and services, the `allow_chained` attribute gives
services blanket permission to perform chained API calls with the token
resulting from the Application credential. This is implemented as follows:

1) If `keystonemiddleware` receives a request that is permitted due to an
   application credential with the `allow_chained` attribute set, it requests a
   service token and adds it to the request's object's headers. Keystone only
   allows setting this `allow_chained` attribute for an application credential
   all capabilities' underlying URL templates have the `allow_chained`
   attribute set to `True`.

2) Follow-up requests issued by the service will then send this service token
   along with the regular token resulting from the application credential.

3) If `keystonemiddleware` encounters an application credential generated token
   with `allow_chained` plus a valid service token it will ignore any
   non-empty capability lists and pass the request to the service as-is.

API Examples
------------

An example creation request for an application credential might look as
follows:

::

    POST /v3/users/{user_id}/application_credentials

.. code-block:: json

    {
        "application_credential": {
            "allow_chained": false,
            "name": "allow-metrics-logs",
            "description": "Allow submitting metrics and logs to Monasca",
            "roles": [
                {"name": "monasca-agent"}
            ]
            "capabilities": [
              {
                "path": "/v2.0/metrics",
                "substitutions": {},
                "type": "POST",
                "url_template": "376a83c4-c6e9-4cdf-b413-ba4880bfda4d"
              },
              {
                "path": "/v3.0/logs",
                "substitutions": {},
                "type": "POST",
                "url_template": "c73beef3-c982-4ed8-86d5-dd362af48614"
              }
            ]
        }
    }

An example creation request (issued by an operator) for a URL template might
look as follows:

::

    POST /v3/capability-templates

.. code-block:: json

    {
        "capability_template": {
            "allow_chained": true,
            "role_id": "0dbbcb80-9d70-4c86-b38a-ae826e501885",
            "path": "/v2.1/servers/**",
            "substitutions": {},
            "service": "67764758-3bdb-462e-babf-537c8fbe7bcd",
            "type": "GET"
        }
    }

Any user may discover the current list of URL through a

::

    GET /v3/capability-templates

In response they will get a list of URL templates:

.. code-block:: json

    [
        {
          "capability_template": {
              "id": "5631dd39-1451-4101-a961-bbc949624b2f",
              "allow_chained": true,
              "role_id": "0dbbcb80-9d70-4c86-b38a-ae826e501885",
              "path": "/v2.1/servers/**",
              "substitutions": {},
              "service": "67764758-3bdb-462e-babf-537c8fbe7bcd",
              "type": "GET"
              }
        },
        {
          "capability_template": {
              "id": "cdfeecfb-752a-4370-9aaf-03751d3645b3",
              "allow_chained": false,
              "role_id": null,
              "path": "/v2.1/servers/a13b634a-dde3-4e5d-bbcb-3c1482bcf6c8",
              "substitutions": {},
              "service": "67764758-3bdb-462e-babf-537c8fbe7bcd",
              "type": "POST"
              }
        },
        {
          "capability_template": {
              "id": "e86584c8-1a1a-4f5d-9da9-da5e265a0423",
              "allow_chained": false,
              "role_id": null,
              "path": "/v2.0/metrics",
              "substitutions": {},
              "service": "1a5e983d-7ac2-4b27-a7a1-caa62a46d82a",
              "type": "POST"
              }
        },
        {
          "capability_template": {
              "id": "8458c208-6a91-4f54-af89-4598b972cd52",
              "allow_chained": false,
              "role_id": null,
              "path": "/v3.0/logs",
              "substitutions": {},
              "service": "f6bd818d-861f-450b-a523-2e1546a06a18",
              "type": "POST"
              }
        }
    ]


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

     (a) Application credential capabilities do not require a `Cambrian
         explosion <https://en.wikipedia.org/wiki/Cambrian_explosion>`_ of
         fine-grained roles (one for every API operation of every OpenStack
         service) that must be managed by an administrator.
     (b) Application credential capabilities does not require any changes to
         existing policy enforcement. Instead, they add an additional check
         that takes place before policy enforcement even comes into play and
         rejects requests early. Not being entangled with policy enforcement
         gives us the freedom to start out with a very basic implementation and
         add features as required later (as opposed to having to be feature
         complete immediately).
     (c) The role check in `keystonemiddleware` targets administrators who want
         to create role profiles for their users, such as "give this user
         read-only access to any services' resources but without letting them
         create new ones". Application credential capabilities on the other
         hand, target OpenStack services and third party applications that only
         need access to a select handful of operations such as "submit SSL
         certificates to the Magnum API for signing".
     (d) Application credential capabilities do not require keystone to be the
         guardian of access control rules, since all the information needed to
         validate access is contained in the token.
     (e) Unlike a policy based check, a capability based check will also work
         for services that do not use `oslo.policy` such as Swift.

3) One implementation detail from the previous section was discussed at length
   at the Rocky PTG: one could have chosen to match for `oslo.policy` targets
   rather than URL paths in the capabilities, which would have been easier in
   some ways. In the end we opted for url paths for the following reasons:

     (a) This is user facing and unlike API paths, policy targets are not
         easily discoverable by the user since there is no documentation on
         them. Moreover, policy targets are not as formalized as APIs and may
         easily change over time, thus breaking existing capabilities.

     (b) URL paths can be rejected in keystonemiddleware, without involving
         `oslo.policy`, leading to a faster failure for unauthorized requests.

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

* URLs in capabilities are user-supplied strings. Care must be taken to
  guard against format string attacks in these if anything beyond character by
  character comparison takes place.

* It might be a good idea to limit the length/number of capability rules per
  API credential to prevent denial of service against the Keystone database (by
  filling it with bogus rules) or the Keystone API (via large validation
  payloads). Another reason to introduce such a limit is the possibility to
  slow down a service by creating application credentials with a large number
  of non-matching capabilities, which can be used to slow down a particular
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
mitigated by limiting the number of capabilities allowed per application
credential.

Other Deployer Impact
---------------------

This change will introduce the following settings for Keystone:

* `[application_credential]/soft_capability_quota` [Default: `5`] This setting
  determines the number of entries allowed in newly created capability lists
  globally. `-1` denotes an unlimited number of entries. Any existing
  application credentials with more capabilities will continue to work.

* `[application_credential]/hard_capability_quota` [Default: `-1`] This setting
  determines the number of entries allowed in capability lists globally.  `-1`
  denotes an unlimited number of entries. Any existing application credentials
  with more capabilities will fail token validation.

Developer Impact
----------------

This change provides developers across all OpenStack services with a means to
create application credentials with fine-grained permissions, allowing them to
delegate access to a user's roles according to the principle of least
privilege.

As far as the application credentials API is concerned, it will be fully
backwards compatible, since specifying capabilities when creating an
application credential is optional: if none are specified, the `capabilities`
attribute will be `None`, leading to no capability checks being performed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * Johannes Grassler <jgr-launchpad@btw23.de> jgr-launchpad

Other contributors:

  * Colleen Murphy <colleen@gazlene.net> cmurphy

  * Adam Young <ayoung@redhat.com> ayoung

Work Items
----------

1. Extend the application credential API and database schema in Keystone to
   allow for receiving and storing capability lists.

2. Implement handling for capabilities in python-keystoneclient and
   python-openstackclient.

3. Extend the Keystone token validation API to capability lists upon
   upon token validation.

4. Implement the endpoint list check in keystonemiddleware.

Dependencies
============

None

Documentation Impact
====================

* The capability related settings for application credentials need to be
  documented in the release notes and the admin guide.

* The URL template "language" outlined in the `Permissible Path Templates`_
  section needs to be documented in the Keystone admin guide.

* Documentation on capabilities needs to be added to the *Application
  Credentials* section of the Keystone user documentation.

References
==========

* Etherpad with original proposal from the Barcelona 2016 summit:
  https://etherpad.openstack.org/p/ocata-keystone-authorization

* Etherpad with refined proposal from the Rocky PTG 2018:
  https://etherpad.openstack.org/p/application-credentials-rocky-ptg

* Spec for securing Monasca metric submission from inside VMs
  https://review.openstack.org/#/c/507110/ (would be greatly simplified by
  having capabilities in application credentials)

* Documentation on Barbican ACLs:
  http://developer.openstack.org/api-guide/key-manager/acls.html

* Documentation on Swift ACLs:
  https://www.swiftstack.com/docs/cookbooks/swift_usage/container_acl.html

* Generating a list of URL patterns for OpenStack services
  http://adam.younglogic.com/2018/03/generating-url-patterns/

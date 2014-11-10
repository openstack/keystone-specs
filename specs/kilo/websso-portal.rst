..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Web Single Sign On Portal
=========================

`bp websso-portal
<https://blueprints.launchpad.net/keystone/+spec/websso-portal>`_

Provide the ability for a user to authenticate via a web browser with an
existing IdP, through a Single Sign-On page. Keystone should be able to create
a token and post it back to a requestor to support Federation protocols.

Since this is a cross-project spec, the blueprint for horizon is:
`bp federated-identity
<https://blueprints.launchpad.net/horizon/+spec/federated-identity>`_

Problem Description
===================

With password based authentication, Horizon has been responsible for carrying
the shared secret between the user and Keystone. Cryptographically secure
authentication mechanisms protect against this, as it leads to various
Man-in-the-middle attack scenarios. If the Horizon server is compromised, it
would grant the attacker access to the password for all users that log in.
When coupled with a corporate directory, this poses a significant risk.

The proper sequence is
* Horizon redirects the browser to Keystone
* Keystone redirects to Discovery Service or default IdP
* The User authenticates via an IdP specific mechanism
* The IdP redirects the browser back to Keystone
* Keystone generates Javascript code to POST a token back to Horizon
* Keystone redirects the user to Horizon.

The Keystone server needs to enable the redirect to the IdP, and accept back
the SAML assertion in order to create the token.

Proposed Change
===============

Horizon would implement a way to choose preferred authentication method.
Technically, a user would be redirected to different route at the Keystone
server.
Keystone will implement additional routes/endpoints that allow for different
forms of login. Initially these will be:

* Federation via SAML
* Federation via OpenID Connect

For SAML and OpenID Connect, the Horizon (or compatible WebUI)  will redirect
to the appropriate protected Keystone URL (depending on the implementation,
``/v3/OS-FEDERATION/websso/{protocol}?origin=https%3A//horizon.example.com``
or ``/v3/auth/websso/{protocol}?origin=https%3A//horizon.example.com``).

It's important to mention that ``{protocol}`` variables must be registered (and
tied to ``identity provider`` object issuing assertions) via the
``OS-FEDERATION`` API.

The algorithm for identifying ``mapping`` to be used:

::

    1) Fetch attribute storing identity provider identifier.
    2) Fetch identity_provider basing on the previous value (return error
    code if no such identity_provider was found).
    3) Fetch protocol value from the URL (return error if no such
    protocol was found tied to such identity_provider).
    4) Fetch mapping id identified by identity_provider and protocol
    ids.

JavaScript that performs a HTTP POST to the original webUI. The URL that
Keystone will POST user back will be stored in the initial URL.

Note, that Horizon will not accept POST requests, so the function that
handles this portion would need to be exempt from CSRF errors.

The proposed work flow is depicted on an ASCII diagram below.

::

    +----------------+      +----------------+      +----------------+
    |                |      |                |      |                |
    |    Horizon     |      |    Keystone    |      |Discover Service|
    |                |      |                |      |     (IdP)      |
    |                |      |                |      |                |
    +---^----+----^--+      +--^---+----^---++      +---^-------+----+
        |    |    |            |   |    |   |           |       |
       (1)  (2a) (5b)         (2b)(3a) (4b)(5a)        (3b)    (4a)
        |    |    |            |   |    |   |           |       |
    +---+----v----+------------+---v----+---v-----------+-------v----+
    |                                                                |
    |                          Web Browser                           |
    |                                                                |
    +----------------------------------------------------------------+

The work flow is as follows:

::

    1) A User with a web browser reaches Horizon

    2a) Horizon issues an HTTP 301 Redirect response pointing the user to
    the Keystone endpoint.

    2b) The browser redirects the user to a Keystone ``websso`` endpoint,
    ``/auth/OS-FEDERATION/websso/{saml2/oidc}`` which is protected by
    saml2 and openid connect Apache plugins respectively.

    3a) Since the user doesn't have an active session, the user will be
    redirected to their default IdP or Discover Service where an appropriate
    IdP can be chosen. Keystone will produce an appropriate <saml2:Request>
    request.

    3b) The browser sends a <saml2:Request> request and authenticates with the
    IdP.  The user also authenticates himself (e.g. with user/password
    credentials)

    4a) Once authentication is successful, the IdP issues a SAML assertion or
    OpenID Connect claim, and redirects user back to the
    ``/auth/OS-FEDERATION/websso/{protocol}``.

    4b) The browser redirect the user back to Keystone again, with the SAML
    assertion or OpenID Connect claim.

    5a) Keystone issues JavaScript code that executes upon loading by the
    browser. The unscoped federated token is included in the HTML form.

    5b) The user is being redirected back to the Horizon webpage, with the
    unscoped federated token in the request.

Alternatives
------------

All of the user interface could be maintained in Horizon, but then sensitive
information is provided beyond the scoped of the Keystone service. Horizon and
any other web UI would need deep integration with the Keystone WebSSO process,
and the two systems would effectively be one tightly coupled system.

Security Impact
---------------

Describe any potential security impact on the system. Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?
* Yes, Tokens.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
* This is a new way to log in to Horizon.

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.
  No.

Note, that in order to avoid phishing and other security related attacks (a
token is being transferred). Keystone needs a mechanism to register a trusted
Horizon URL (possibly via an API or a config setting). This will ensure that
the tokens are not posted back to an untrusted party.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This is primarily a visual feature. No impact on python-keystoneclient.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

* This feature will be enabled based on changes to the Apache HTTPD
  configuration file. It should not require any additional editing by end
  deployers afterwards.

* These changes should not break current continuous deployment.

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* ayoung (Adam Young <ayoung@redhat.com>)

Other contributors:

* stevemar (Steve Martinelli <stevemar@ca.ibm.com>)
* marek-denis (Marek Denis <marek.denis@cern.ch>)
* jose-castro-leon (Jose Castro Leon <jose.castro.leon@cern.ch>)
* tqtran (Thai Tran <tqtran@us.ibm.com>)

Work Items
----------

`django_openstack_auth`

* Add a new WEBSSO_ENABLED option
* New button that allows a user to select a federation log-in
* New settings to support a single IdP
  * WEBSSO_IDP_URL # the URL of the SSO of the IdP
  * WEBSSO_IDP_ID # the Keystone Identity provider ID
  * WEBSSO_PROTOCOL_ID # the Keystone protocol ID
* Support login with an unscoped token
* Perform a validate token call

`keystone`

* Add new endpoints: either
  ``/v3/auth/websso/{protocol}?origin=https%3A//horizon.example.com`` or
  ``/v3/OS-FEDERATION/websso/{protocol}?origin=https%3A//horizon.example.com``
  (protected by mod_shib, mod_oidc respectively), handling websso requests for
  federated protocols (SAML2 or OpenID Connect).
* Add logic associating an issuer of an assertion/claim with an
  identity_provider object registered in Keystone's backend (via remote_id
  attribute).
* Changes to middleware to allow for HTML in the response.


Dependencies
============

This specification will depend on work delivered in `bp idp-id-registration
<https://blueprints.launchpad.net/keystone/+spec/idp-id-registration>`_

Documentation Impact
====================

Configuration manual will have to explain how to set up the Apache server to
expose a discovery service


References
==========

CERN Patch to Keystone support WebSSO
https://github.com/cernops/keystone/commit/66dabd94b4ad32abca171cef9192210fec289235

CERN Patch to Django-Openstack-Auth to support WebSSO
https://github.com/cernops/django_openstack_auth/commit/b7e5b28a83a88b259bfaddbd754c70e1bb420447

Patchset adding ``remote_id`` attribute to the ``identity_provider`` objects
https://review.openstack.org/#/c/142743/

Discovery Service specification:
http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-idp-discovery.pdf

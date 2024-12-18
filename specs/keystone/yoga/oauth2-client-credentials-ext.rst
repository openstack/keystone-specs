..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
OAuth2.0 Client Credentials Grant Flow Support
==============================================


Provide the ability for users to delegate roles to third-party clients via the
Client Credentials Grant in `RFC6749 OAuth 2.0 Authorization Framework`
[#oauth2_specification]_. Requires v3.0+ of the Identity API. An
OAuth-derived token allows third-party clients to access protected
resources created by a user.

Problem Description
===================

In the current Keystone implementation, OAuth2.0 is not supported.

However, OpenStack Tacker, which provides NFV orchestration APIs, needs
OAuth2.0 support to meet a well-known NFV standard referred to as ETSI NFV. In
the API specification of ETSI NFV [#nfv-sol013]_, only OAuth2.0 Client
Credentials Grant flow is allowed for the API client authorization.  To meet
this requirement, we need OpenStack to have OAuth2.0 authorization server
functionality.

Proposed Change
===============

The proposed change is to allow users to optionally use an OAuth2.0 Client
Credentials Grant flow to authorize an API client.  In order to realize this,
we implement an OAuth2.0 authorization server as an extension of keystone.

This server supports the following APIs involved in access token management.

* Access Token API

  * Create Access Token

.. warning::
  Note that, according to RFC6749 [#oauth2_specification]_, HTTPS must be
  enabled in the authorization server since some requests include sensitive
  information, e.g., a client secret, in plain text.

.. warning::
   Note that the OAuth 2.0 APIs described in this document use the Application
   Credentials [#application_credentials]_ as its backend. Thus, an OAuth2.0
   user must be assigned an appropriate role to access all Application
   Credentials APIs


Terminology
-----------

- *User:* The end-users who use Identity API service, the entity whose role(s)
  will be delegated, and the entity that registers Clients.

- *Client:* An application making protected resource requests on behalf of the
  User. The credentials of the client are created though the Application
  Credentials API [#application_credentials]_.

- *Access Token:* A token used by the Client to make protected resource
  requests with the delegated roles.

OAuth2.0 Client Credentials Grant Flow
--------------------------------------

.. image:: _images/seqdiag-4d0fa7286c9f95db7c9aab083def7f30064925f6.png

The flow consists of the following steps as illustrated in the above sequence:

#. An Identity API service User creates an application credential.

#. The Client authenticates with the authorization server on the Keystone and
   requests a new Access Token.

#. The Client uses the Access Token to make requests for OpenStack Service
   APIs.

#. Keystone Middleware validates the Access Token in an API request to obtain
   its metadata and validity, and forwards the request to the OpenStack service
   if the token is active.

API Resources
-------------


Access Token API
----------------


Create Access Token
~~~~~~~~~~~~~~~~~~~

::

    POST /identity/v3/OS-OAUTH2/token

Request:

::

  Host: server.example.com
  Authorization: Basic NzkxZDVlZDI2MjAxNDE4NWI4NTRlZjJhZGUwZGM0NWE6SkRKaUpEQTBKRXhpVnpBM2JtMUVaazVRTUhOWlpuSmxZMUJXZVM1UE1qY3dNR3hZZFROc1JtbG1jVE5wY1Vka2NtNVdkVkZ6TlhwNGFHVlQ=
  Content-Type: application/x-www-form-urlencoded

  grant_type=client_credentials

Response:

::

  HTTP/1.1 200 OK
  Content-Type: application/json;charset=UTF-8
  Cache-Control: no-store
  Pragma: no-cache

  {
    "access_token": "gAAAAABhi1cMynG89h8t6TJrxNiZuNzjcIUIxNctoVfuqTw7BpUedLKxjPymClVEnj9GhIT5u2mpjaJATlEAtaa3D6_t8jk_fV-mqo2IUlsmTPTnMwkcjh5FSHQVRdqvDxgY3nSqLA_Hfv-zPmjS5KWX3hmyDE5YWO1ztX6QNVQb4wTPyNL1-7I",
    "token_type": "Bearer",
    "expires_in": 3600
  }

Error response:

::

  HTTP/1.1 401 Unauthorized
  Content-Type: application/json
  WWW-Authenticate: Keystone uri="http://keysone.identity.host/identity/v3/auth/clients"
  Cache-Control: no-store
  Pragma: no-cache

  {"error": “invalid_client", “error_description": “The client_id is not found or client_secret is invalid."]}


The Client requests an Access Token with its credential. The Identity service
issues a new access token if the credential is valid. Otherwise returns an
error response. ``Authorization`` with the ``Basic`` field in the response
contains the Base64 encoding of Application Credentials ID and password joined
by a single colon.

The Keystone Middleware obtain the metadata of Access Token via the Identity
API ``/identity/v3/auth/tokens``.

Note that, according to RFC6749 [#oauth2_specification]_, the "bearer" token
type defined in RFC6750 [#bearer_token]_ is utilized for including the access
token string in the API request.  The Keystone Middleware has to obtain an
access token from a request with the Authorization header.

::

   GET /resource HTTP/1.1
   Host: server.example.com
   Authorization: Bearer gAAAAABhi1cMynG89h8t6TJrxNiZuNzjcIUIxNctoVfuqTw7BpUedLKxjPymClVEnj9GhIT5u2mpjaJATlEAtaa3D6_t8jk_fV-mqo2IUlsmTPTnMwkcjh5FSHQVRdqvDxgY3nSqLA_Hfv-zPmjS5KWX3hmyDE5YWO1ztX6QNVQb4wTPyNL1-7I

The Keystone Middleware updates request headers with the metadata only if a
token is valid. If a token is invalid or an error response is returned, it
rejects a request and returns ``401 Unauthorized``.  The Keystone Middleware
uses "Authentication and token management API"
[#authentication_and_token_management]_ to validate and get token metadata.

Alternatives
------------

None

Security Impact
---------------

* During the Client Credentials Grant flow, some sensitive values are sent in
  plain text. Thus, Keystone using this feature must enable HTTPS.

* This code will be managing the negotiation between keystone and third-party
  applications. However, the backend of this function is Application
  Credentials which is already implemented in keystone. We would implement only
  the logic needed on top of this functionality.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

Configuration of Authorization Server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As we would provide OAuth2.0 functionality as an extension, it does not affect
the existing deployers.  A deployer can enable this feature by adding
configuration blocks.  The following is an example of the configuration.

::

  [oauth2]
  driver = sql

  [auth]
  methods = external,password,token,oauth2


Configuration of Keystone Middleware
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To use OAuth2.0 access token, a deployer has to configure Keystone Middleware
by changing ``[filter:authtoken]`` in ``/etc/tacker/api-paste.ini`` as shown
below.  If ``paste.filter_factory`` is
``keystonemiddleware.oauth2_token:filter_factory``, the Keystone Middleware
expects to find a token with the ``Authorization`` header, whereas if
``paste.filter_factory`` is ``keystonemiddleware.oauth2_token:filter_factory``,
the Keystone Middleware expects to find a token with the ``X-Auth-Token``
header.

::

  [filter:authtoken]
  paste.filter_factory = keystonemiddleware.oauth2_token:filter_factory


Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * Hiromu Asahina (h-asahina) <hiromu.asahina.az@hco.ntt.co.jp>

Other contributors:
  * Yusuke Niimi <niimi.yusuke@fujitsu.com>
  * Keiichiro Yamakawa <yamakawa.keiich@fujitsu.com>

Work Items
----------

* Add new REST API endpoints to keystone for OAuth2.0 Client Credentials Grant
  Flow.
* Make changes in keystoneauth to support the authorization with OAuth2.0
  access token.
* Add unit tests for the new endpoints.
* Add a new AuthProtocol to handle "bearer" token type to the Keystone
  Middleware
* Add unit tests for the new AuthProtocol.
* Change API Keystone documentation.
* Change API Keystone Middleware documentation.

Dependencies
============

None

Documentation Impact
====================

* We would need to update the user API docs and Authentication Mechanisms.
* We would need to update the user API docs and Middleware Architecture.

References
==========

.. [#oauth2_specification] https://tools.ietf.org/html/rfc6749
.. [#bearer_token] https://datatracker.ietf.org/doc/html/rfc6750
.. [#nfv-sol013]
   https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/013/02.06.01_60/gs_nfv-sol013v020601p.pdf
.. [#authentication_and_token_management]
   https://docs.openstack.org/api-ref/identity/v3/index.html?expanded=validate-and-show-information-for-token-detail#authentication-and-token-management
.. [#application_credentials]
   https://docs.openstack.org/api-ref/identity/v3/index.html#application-credentials

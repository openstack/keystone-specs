..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Support OAuth 2.0 Mutual-TLS
============================

Provide the option for users to proof-of-possession of OAuth2.0 access token
based on `RFC8705 OAuth 2.0 Mutual-TLS Client Authentication and
Certificate-Bound Access Tokens` [#oauth2_specification]_ (hereafter OAuth2.0
mutual TLS). Requires v3.0+ of the Identity API and assumes TLS 1.2 for mutual
TLS. This method will protect Keystone Identity and other OpenStack services
from spoofed OAuth clients.

Problem Description
===================

OAuth2.0 Client Credentials Grant implemented in Keystone uses id/password to
request an access token. However, passwords can be defeated using various
techniques such as dictionary attacks. In addition, the authorization server
(i.e., Keystone) cannot detect the spoofed password and more importantly it
cannot detect a spoofed access token. For this reason, OpenStack Tacker, which
provides NFV orchestration APIs, needs OAuth2.0 mutual TLS support to meet a
well-known NFV standard referred to as `ETSI NFV`` [#nfv_sol013]_. As described
in that standard, such attacks are far more difficult when both sides have to
authenticate with TLS certificates, i.e., `mutual TLS` [#oauth_mtls]_.

Proposed Change
===============

The proposed change is to support OAuth2.0 mutual TLS. This method ensures that
only the party in possession of the private key corresponding to the X.509 TLS
client certificate can utilize the token to access the protected resources.

For this to happen, the following items have to be implemented.

* Add client certificate-based authentication using mutual TLS to Keystone

* Add option to create certificate-bound access tokens to Keystone identity and
  ability to issue certificate bound to access tokens

* Add verification of the client certificate to Keystone middleware

* Add confirmation of proof-of-possession of an access token to Keystone
  middleware

* Add configuration parameters needed to use mutual TLS (e.g., client
  certificate, CA certificate) to keystoneauth

.. note::

   In order to use mutual TLS, a client has to generate a client certificate
   from its public/private key pair. The client certificate must be an X509
   certificate signed by a private/public Certificate Authority (CA) whose
   certificate must be available on both client and Keystone. These files
   should be externally generated.

.. note::

  In order to use OAuth2.0 mutual TLS, mutual TLS should be configured on
  Keystone. Although enabling mutual TLS also on the other service endpoints
  makes system more secure, the details of such configuration are out of the
  scope of the present document.

Terminology
-----------

- *Client:* An application making protected resource requests. The Client can
  be a user created by Identity API whose attributes must identical with the
  subject DN of the client certificate.

- *Access token:* A token used by the Client to make protected resource
  requests with the delegated roles.

- *Client certificate:* A certificate used by the Client to proof-of-possession
  of an access token

OAuth2.0 mutual TLS Flow
------------------------

.. image:: _images/seqdiag-ae6acb56f902293e5b492939b97ba256bbf31fec.png

The flow consists of the following steps as illustrated in the above sequence:

#. Client requests a new access token to Keystone

   A Client authenticates with Keystone and requests a new access token. For
   the authentication, the Client uses its client certificate as a credential.
   The Client must be registered as a Keystone user [#user_api]_ in advance and
   its attributes such as ``username``, ``project`` and ``domain``, must be
   identical with the subject of the client certificate.

#. Web server for Keystone verifies the validity of the client certificate

   Upon obtaining the client certificate from the request, a web server for
   Keystone (i.e., Apache) verifies a trust chain of the certificate and the
   possession of the private key that corresponds to the certificate, then
   continues to the next step only if it's valid (e.g., it was signed by a
   known, trusted CA and the client presents the possession of the private
   key), otherwise rejects the request.

#. Keystone issues a certificate-bound access token to Client

   Keystone creates an access token with the confirmation of whether the
   subject DN of the client certificate matches the user attributes. The
   mapping rules between the subject DN of client certificates and user
   attributes are described by the OS-FEDERATION API [#federation_mapping]_. If
   the token is successfully created, Keystone binds the certificate to the
   token and issues the token to the Client as an OAuth2.0 access token.

#. Client makes API requests using the issued access token over mutual TLS

   Client makes an API request with an issued access token over the mutual TLS.
   The mutual TLS is triggered by the server, and thus the requirements for the
   client is just to be capable of setting its certificate to API requests.

#. Web server for OpenStack Service verifies the validity of the client
   certificate

   Upon obtaining the client certificate from the request, a web server for
   OpenStack Service verifies a trust chain of the certificate and the
   possession of the private key that corresponds to the certificate, then
   continues to the next step only if it's valid, otherwise rejects the
   request.

#. Keystone Middleware verifies the validity of the access token

   Keystone verifies the validity of the access token contained in the request
   by checking that the certificate thumbprint in metadata matches the
   thumbprint of the certificate in mutual TLS and the access token is not
   expired. The thumbprint and the status of token can be obtained from
   Keystone Authentication and token management API
   [#authentication_and_token_management]_.

The following existing API should be modified to handle requests using
OAuth2.0 mutual TLS.

* Access Token API

  * Create Access Token

Also, to serve client certificate thumbprints from Keystone to Keystone
Middleware, the response of the following existing API has to be modified.

* Keystone Authentication and token management API

  * Validate and Show Information for Token

API Resources
-------------

Access Token API
----------------

Access Token API has already been implemented. Therefore, only the changes
required to handle OAuth2.0 mutual TLS requests are described in the present
document.

Create Access Token
~~~~~~~~~~~~~~~~~~~
::

  POST /identity/v3/OS-OAUTH2/token

Request:

::

  Host: server.example.com
  Content-Type: application/x-www-form-urlencoded
  grant_type=client_credentials

  client_id=jFtpUlndpRGaAHuh9TsP3wtj

The client authentication in the existing patches uses
``client_id``/``client_secret`` as credential, whereas OAuth2.0 mutual TLS
assumes a client certificate is used as a credential. This API will be changed
to authenticate clients with their client certificate to handle both cases.
When a request is sent over the mutual TLS, a client is successfully
authenticated only if the client certificate is valid and the subject DN of the
certificate matches the user attributes. Note that the validity of client
certificate can be checked by the general process of mutual TLS. It is also
noted that the mapping rules between the subject DN and user attributes can be
configured by a the mapping in the OS-FEDERATION API.

The following is an example of mapping rules. This example contains two mapping
rules for different CAs (i.e., ``root_a.openstack.host`` and
``root_b.openstack.host``). When the CN name of the issuer of the client
certificate is ``root_a.openstack.host``, the client certificate must contain
the five fields defined in ``remote`` block of the rule (i.e.,
``SSL_CLIENT_SUBJECT_DN_CN``, ``SSL_CLIENT_SUBJECT_DN_UID``,
``SSL_CLIENT_SUBJECT_DN_EMAILADDRESS``, ``SSL_CLIENT_SUBJECT_DN_O``, and
``SSL_CLIENT_SUBJECT_DN_DC``) and those fields must match an existing Keystone
user's attributes specified in ``local`` block of the rule (i.e., ``name``,
``id``, ``email``, ``domain name``, ``domain id``). If the ``remote`` fields do
not match corresponding ``local`` fields, Keystone doesn't issue a token.
Likewise, when the CN name is ``root_b.openstack.host``, Keystone requires a
client certificate that have two fields defined in ``remote`` block and
requires the values of those fields match the values of an existing Keystone
user attributes defined in the ``local`` block.

.. code-block:: json

    [
      {
        "local": [
          {
            "user": {
              "name": "{0}",
              "id": "{1}",
              "email": "{2}",
              "domain": {
                "name": "{3}",
                "id": "{4}"
              }
            }
          }
        ],
        "remote": [
          {
            "type": "SSL_CLIENT_SUBJECT_DN_CN"
          },
          {
            "type": "SSL_CLIENT_SUBJECT_DN_UID"
          },
          {
            "type": "SSL_CLIENT_SUBJECT_DN_EMAILADDRESS"
          },
          {
            "type": "SSL_CLIENT_SUBJECT_DN_O"
          },
          {
            "type": "SSL_CLIENT_SUBJECT_DN_DC"
          },
          {
            "type": "SSL_CLIENT_ISSUER_DN_CN",
            "any_one_of": [
                "root_a.openstack.host"
           ]
          }
        ]
      },
      {
        "local": [
          {
            "user": {
              "id": "{0}",
               "domain": {
                "id": "{1}"
              }
            }
          }
        ],
        "remote": [
          {
            "type": "SSL_CLIENT_SUBJECT_DN_UID"
          },
          {
            "type": "SSL_CLIENT_SUBJECT_DN_DC"
          },
          {
            "type": "SSL_CLIENT_ISSUER_DN_CN",
            "any_one_of": [
                "root_b.openstack.host"
           ]
          }
        ]
      }
    ]


If the authentication is successful, Keystone binds the client certificate to
the access token. Assuming the fernet token is used as an access token, this
can be done by adding the thumbprint of a client certificate into the payload
of the fernet token.

Keystone Authentication and token management API
------------------------------------------------

Keystone Authentication and token management API has already been implemented.
Therefore, only the changes required to handle OAuth2.0 mutual TLS are
described in the present document.

Validate and Show Information for Token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /v3/auth/tokens

Response:

::

  HTTP/1.1 200 OK
  Content-Type: application/json

  {
    "token": {
      "audit_ids": [
          "mAjXQhiYRyKwkB4qygdLVg"
      ],
      "catalog": [
        {
          "type": "identity",
          "name": "keystone",
          "endpoints": [
            {
              "region": "RegionOne",
              "adminURL": "http://10.10.1.100/identity",
              "publicURL": "http://10.10.1.100/identity"
            }
          ]
        }
      ],
      "expires_at": "2015-11-07T02:58:43.578887Z",
      "is_domain": false,
      "issued_at": "2015-11-07T01:58:43.578929Z",
      "methods": [
            "password"
      ],
      "project": {
        "domain": { "id": "default", "name": "Default" },
        "id": "f2796050af304441b5f1eabecb33e808",
        "name": "service"
      },
      "roles": [
        {
          "description": null,
          "domain_id": null,
          "id": "d229bd3566fe4abe96a5d02c211e2f10",
          "name": "admin",
          "options": { "immutable": true }
        },
        {
          "description": null,
          "domain_id": null,
          "id": "c9b1b27aeff440959db75bdc91dd8a84",
          "name": "member",
          "options": { "immutable": true }
        }
      ],
      "user": {
        "domain": { "id": "default", "name": "Default" },
        "id": "da0e3ae640584af98c015343b0552ec0",
        "name": "client",
        "password_expires_at": null
      }
      "OS-OAUTH2": {
        "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg2"
      }
    }
  }

The difference from the current API response is to add the ``OS-OAUTH2`` field
that contains the client certificate subject DN corresponding to the access
token in ``x5t#S256`` field. This field is added only when the token is issued
by OAuth2.0 access token API and the response status is 200. The other fields
and error responses are the same as the current API implementation.

The Keystone Middleware sends requests to this modified ``Keystone
Authentication and token management API`` with the access token contained in an
API request in order to check that the certificate thumbprint in metadata
matches the thumbprint of the certificate presented during the mutual TLS and
the status of the access token is not expired.

If the token passes the validation, the Keystone Middleware updates request
headers with the metadata. If a token is invalid or an error response is
returned, it rejects a request and returns ``401 Unauthorized``.

According to RFC6749, the "bearer" token type defined in RFC6750
[#bearer_token]_ is utilized for including the access token string in the API
request. The Keystone Middleware has to obtain an access token from a request
with the Authorization header. An example of such a request is shown below.

::

   GET /resource HTTP/1.1
   Host: server.example.com
   Authorization: Bearer f69c9fb6947c47329b8955d629ac5722

Alternatives
------------

Verification of Token Validity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can avoid modifying the existing Keystone Authentication and token
management API, by adding a new Token introspection API [#token_introspection]_
as follows.

::

    POST /identity/v3/auth/OS-OAUTH2/introspect

Request:

::

  Host: server.example.com
  Content-Type: application/x-www-form-urlencoded

  client_id=jFtpUlndpRGaAHuh9TsP3wtj&token=f69c9fb6947c47329b8955d629ac5722&token_type_hint=access_token

Response:

::

  HTTP/1.1 200 OK
  Content-Type: application/json

  {
    "active": true,
    "cnf":{
      "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg2"
    }
    "audit_ids": [
        "mAjXQhiYRyKwkB4qygdLVg"
    ],
    "catalog": [
      {
        "type": "identity",
        "name": "keystone",
        "endpoints": [
          {
            "region": "RegionOne",
            "adminURL": "http://10.10.1.100/identity",
            "publicURL": "http://10.10.1.100/identity"
          }
        ]
      }
    ],
    "expires_at": "2015-11-07T02:58:43.578887Z",
    "is_domain": false,
    "issued_at": "2015-11-07T01:58:43.578929Z",
    "methods": [
          "password"
    ],
    "project": {
      "domain": { "id": "default", "name": "Default" },
      "id": "f2796050af304441b5f1eabecb33e808",
      "name": "service"
    },
    "roles": [
      {
        "description": null,
        "domain_id": null,
        "id": "d229bd3566fe4abe96a5d02c211e2f10",
        "name": "admin",
        "options": { "immutable": true }
      },
      {
        "description": null,
        "domain_id": null,
        "id": "c9b1b27aeff440959db75bdc91dd8a84",
        "name": "member",
        "options": { "immutable": true }
      }
    ],
    "user": {
      "domain": { "id": "default", "name": "Default" },
      "id": "da0e3ae640584af98c015343b0552ec0",
      "name": "client",
      "password_expires_at": null
    }
  }

Response (expired token):

::

  HTTP/1.1 200 OK
  Content-Type: application/json

  {
    "active": false,
  }


Error response:

::

  HTTP/1.1 401 Unauthorized
  Content-Type: application/json
  WWW-Authenticate: Keystone uri="http://keysone.identity.host/identity/v3/users/{user_id}/application_credentials"
  Cache-Control: no-store
  Pragma: no-cache

  {
    "error": "invalid_client",
    "error_description": "The client_id is not found or client_certificate is invalid."
  }

The Keystone Middleware requests the Token Introspection with the access token
contained in an API request. Keystone returns the metadata corresponding to the
token, such as the thumbprint of client certificate, the service catalog, user
Id, token validity, etc, if the credential is valid. Otherwise, it returns an
error response.

The Keystone receiving this API request has to obtain the token metadata
through the two steps:

#. Validate and obtain information for token via Keystone API
   `/v3/auth/tokens`
#. Retrieve the thumbprint of a client certificate subject DN

If the token passes the validation and it's not expired, Keystone set the JSON
object merging the required field in `RFC7662: OAuth 2.0 Token Introspection`
(i.e., `active`) into the response from `/v3/auth/tokens`. Specifically, the
fields in the response are considered the `extension_field`. The client
certificate subject DN is set to ``x5t#S256`` field. In the cases where the
token is expired and authentication fails, Keystone returns 200 responses with
the body indicating `active: false` and 401 error response, respectively.

The Keystone Middleware updates request headers with the metadata only if a
token is valid. If a token is invalid or an error response is returned, it
rejects a request and returns ``401 Unauthorized``. The validity of a token is
determined by the value of the ``active`` field in a response, i.e., a token is
valid if the value is ``true``, and invalid if the value is ``false``.

Another alternative is to use JWT including the thumbprint of a client
certificate as a field (See `RFC8705: 3.1 JWT Certificate Thumbprint
Confirmation Method` [#oauth2_mtls_jwt]_). In this case, we can omit the token
introspection API.

Creation of Certificate-bound Access Tokens
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As an alternative of the fernet token, we can use JWT token which is already
supported by Keystone if we can't modify the fernet token contents freely. In
that case, the thumbprint of a client certificate or ``credential Id`` which
will be used to retrieve the certificate from the credentials table is included
as a field of JWT (See `RFC8705: 3.1 JWT Certificate Thumbprint Confirmation
Method`).

Security Impact
---------------

No negative impact on the Keystone security as using mutual TLS makes OAuth2.0
safer.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

End users have to configure their clients to use the feature described in the
present specification. For this reason, appropriate user documents needs to be
added.

Performance Impact
------------------

To store the thumbprint, the fernet token size will be slightly increased. This
might increase delay, computational cost, etc. However, in general, the
thumbprints are created by a hash function like SHA256, their size is
negligibly small.

Other Deployer Impact
---------------------

Configuration of Authorization Server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As we would provide OAuth2.0 functionality as an extension, it doesn't affect
the existing deployers. A deployer can enable this feature by adding
configuration blocks. The following is an example of the configuration.

::

  [oauth2]
  driver = sql
  token_endpoint_auth_method = tls_client_auth

  [auth]
  methods = external,password,token,oauth2


Configuration of Keystone Middleware
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To use OAuth2.0 access token, a deployer has to configure Keystone Middleware
in OpenStack services by changing ``[filter:authtoken]`` in, for example,
``/etc/tacker/api-paste.ini`` as shown below. If ``paste.filter_factory`` is
``keystonemiddleware.oauth2_mtls_token:filter_factory``, the Keystone
Middleware expects to mutual TLS is used for API request and the access token
in the request is bound to the client certificates.

::

  [filter:authtoken]
  paste.filter_factory = keystonemiddleware.oauth2_mtls_token:filter_factory

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

* Add client certificate-based authentication using mutual TLS to Keystone

* Add option to create certificate-bound access tokens to Keystone identity and
  ability to issue certificate bound to access tokens

* Add verification of the client certificate to Keystone middleware

* Add confirmation of proof-of-possession of an access token to Keystone
  middleware

* Add configuration parameters needed to use mutual TLS (e.g., client
  certificate, CA certificate) to keystoneauth

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
.. [#nfv_sol013] https://www.etsi.org/deliver/etsi_gs/NFV-SOL/001_099/013/02.06.01_60/gs_nfv-sol013v020601p.pdf
.. [#oauth_mtls] https://datatracker.ietf.org/doc/html/rfc8705
.. [#user_api] https://docs.openstack.org/api-ref/identity/v3/#users
.. [#federation_mapping] https://docs.openstack.org/api-ref/identity/v3-ext/index.html?expanded=create-a-mapping-detail#mappings
.. [#authentication_and_token_management]
   https://docs.openstack.org/api-ref/identity/v3/index.html?expanded=validate-and-show-information-for-token-detail#authentication-and-token-management
.. [#bearer_token] https://datatracker.ietf.org/doc/html/rfc6750
.. [#token_introspection] https://datatracker.ietf.org/doc/html/rfc7662
.. [#oauth2_mtls_jwt] https://datatracker.ietf.org/doc/html/rfc8705#section-3.1

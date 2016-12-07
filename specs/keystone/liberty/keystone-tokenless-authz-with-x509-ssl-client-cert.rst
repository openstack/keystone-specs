..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================
Tokenless Authorization with X.509 Client SSL Certificate
=========================================================

`bp keystone-tokenless-authz-with-x509-ssl-client-cert
<https://blueprints.launchpad.net/keystone/+spec/
keystone-tokenless-authz-with-x509-ssl-client-cert>`_

Support tokenless authorization with X.509 client SSL certificate to improve
security and simplify deployment.

See `X.509 <http://en.wikipedia.org/wiki/X.509>`_ for more information on X.509
certificates.

Problem Description
===================

In a typical enterprise deployment, SSL client certificate authentication is
required for services talking to Keystone for things such as token validation
and resource lookup. For example, we configure a client certificate for the
``auth_token`` middleware running at the service to establish the SSL client
certificate authentication session with Keystone when validating a user token.
In this case, there's no need for the service to take the extra unnecessary
step to obtain a service token in order to authorize the token validation call.
We can effectively identify the service by its X.509 SSL client certificate.
With the X.509 SSL client certificate attributes, coupled with the scope
information conveyed in the request headers, we can directly create the
authorization ``credential`` by looking up the necessary identity data without
having to issue a token. The authorization ``credential`` is successfully
created if all of the following conditions are true:

1. A SSL client certificate authentication session is successfully established
   between the service and Keystone. This implies the client certificate is
   valid.
2. The certificate meets our filter requirements. i.e. the certificate issuer
   matches one of the issuers that we trust.
3. The attributes from the client certificate map to an existing Keystone user.
4. The Keystone user account is enabled.
5. Scope information is conveyed in the request header (i.e. X-Project-Id)
6. User has one or more roles for the given scope.

The benefits of tokenless authorization with X.509 SSL client certificate are:

* Security - no need to expose the service user password in the conf files.
  Also, this has the effect of non-bearer token as SSL client must possess the
  corresponding private key.

* Simplicity - no need for password. The same certificate can be used for both
  authentication and secured communication.

* Flexibility - this mechanism is not limited to X.509 certificate auth. It can
  be expended to support other external auth mechanisms via different mappings.
  For example, if Kerberos is to be used and the Apache front end populates
  the WSGI request environment in the exactly same manner, all we have to do is
  to create a new identity provider and its corresponding mapping. No code
  changes are necessary.

Proposed Change
===============

This is accomplished by implementing a middleware filter which builds the
authorization ``credential`` based on the certificate attributes and scope
information from the request headers. Authorization ``credential`` is
a Python dictionary which represents the ``credential`` used in Oslo policy
enforcement.

Since we are using Apache mod_ssl to handle the SSL requests, Apache will
convey the client certificate information in the WSGI request environment.
For example::

    SSL_CLIENT_S_DN
    SSL_CLIENT_S_DN_CN
    SSL_CLIENT_S_DN_O
    SSL_CLIENT_S_DN_OU
    SSL_CLIENT_I_DN

See `<http://httpd.apache.org/docs/current/mod/mod_ssl.html>`_ for more details.

This middleware shall use the mapping functionality, implemented for
Federation, to formulate the Keystone identity from the SSL environment
variables. Each issuer (i.e. certification authority) would constitute an
identity provider (IdP). There would be exactly one mapping per IdP. The
IdP ID would be the SHA256 hash of the issuer DN.

In addition, this middleware shall accept the following configurable option,
defined in ``[auth]`` configration section in keystone.conf, to further
filter the certificates that are allowed to participate in tokenless
authorization::

    trusted_issuers = [<list of trust issuer DNs>]

If ``trusted_issuers`` is absent, no certificates will be allowed in tokenless
authorization.

Furthermore, this middleware shall expect the following request headers to
convey the scope information::

    HTTP_X_PROJECT_ID
    HTTP_X_PROJECT_NAME
    HTTP_X_PROJECT_DOMAIN_ID
    HTTP_X_PROJECT_DOMAIN_NAME
    HTTP_X_DOMAIN_ID
    HTTP_X_DOMAIN_NAME

Absence of the scope headers is equivalent to an unscoped token. Notice that
only one scope can be specified for a given request. For example, if both
project and domain scope are specified, that would constitute an error.

Alternatives
------------

Use the existing mechanism, which is to use the service token for
authorization.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

This mechanism has the effect of non-bearer token as the SSL client must
possess the corresponding private key for its SSL certificate.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

In order to use this functionality, service must have a valid X.509 client SSL
certificate. However, certificate management is outside the scope of this spec.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

This feature only works in conjunction with Apache mod_ssl or mod_nss as
standalone eventlet-based Keystone does not parse or convey X.509 client
SSL certificate attributes in the request environment. Therefore, Keystone
must be running in Apache mod_wsgi.

Notice that if the deployment terminates SSL in a load balancer, then the load
balancer must be configured to forward the SSL client certificate.

In order for Apache mod_ssl to convey the SSL client certificate information in
the request environment, the `SSLOptions` directive must contains `+StdEnvVars`
and the `SSLUserName` directive must be set to a valid SSL requirement
environment attribute.

For example::

    SetEnv AUTH_TYPE SSL
    SSLOptions +StdEnvVars
    SSLUserName SSL_CLIENT_S_DN_CN

.. NOTE::

    SSLUserName directive must not be used with +FakeBasicAuth option.
    For more details, please refer to
    `Apache mod_ssl <http://httpd.apache.org/docs/current/mod/mod_ssl.html>`_
    Also, notice the environment variable ``AUTH_MECHANISM``, this is used to
    determine which mapping to use in case we support other protocols in the
    future.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  guang-yee (gyee)

Other contributors:
  None

Work Items
----------

1. Implement new middleware to create auth context from X.509 certificate
   attributes for tokenless authorization.
2. Update auth_token middleware to make service token optional.


Dependencies
============

This feature only works in conjunction with Apache mod_ssl or mod_nss.
Therefore, Keystone must be running in Apache mod_wsgi.


Testing
=======

There will be unit tests. For integration tests, we may need both X.509
certificate management capability (i.e. Barbican + DogTag) and Apache enabled
in Jenkins.

Documentation Impact
====================

1. Update Keystone configuration doc on how to use the new middleware.
2. Update Keystone middleware configuration doc on how to use SSL client
   certificate instead of service token.

References
==========

* `Apache mod_ssl <http://httpd.apache.org/docs/current/mod/mod_ssl.html>`_
* `generic mapping <http://specs.openstack.org/openstack/keystone-specs/specs/juno/generic-mapping-federation.html>`_
* `X.509 Certificate <http://en.wikipedia.org/wiki/X.509>`_

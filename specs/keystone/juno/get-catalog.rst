..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Stand alone service catalog
===========================

`bp get-catalog <https://blueprints.launchpad.net/keystone/+spec/get-catalog>`_

This spec introduces a stand alone v3 API request to fetch an authenticated
service catalog, for use with tokens that do not contain a catalog (those
produced using the ``?nocatalog`` query parameter).

Problem Description
===================

PKI tokens are large, even when compression is applied, largely due to the size
of the service catalog that can be included. One way to address the issue is
for the client to request tokens on Identity API v3 using the ``?nocatalog``
query parameter, which removes the service catalog from the token itself.
Unfortunately, this then breaks clients and services that expect a service
catalog to appear in the token.

Proposed Change
===============

On the service-side, we need an explicit API call to retrieve an authenticated
service catalog, such as ``GET /v3/auth/catalog``. This would return the exact
contents of a token's ``catalog`` object, with no modification other than
perhaps ``links``, as is convention in the Identity v3 API. The goal is that
the object should be usable by all existing clients of the v3 service catalog.

``python-keystoneclient`` must be able to make the new API call (perhaps
lazily) when the catalog is accessed.

``keystonemiddleware.auth_token`` must be able to make the new API call to
populate the ``X-Service-Catalog`` header if a token does not contain a
catalog.

Alternatives
------------

Token response bodies could also contain a service catalog that is not encoded
into the PKI token. While this provides a solution for end users, the downside
of this option is that services are still left with tokens lacking a catalog,
and have no means to retrieve one.

Security Impact
---------------

While this introduces a new authenticated API resource, it does not expose any
data that would have been previously hidden from end users.

It's worth noting that the contents of the service-side catalog could change
after the token was issued. Remote services should therefore not expect a
static catalog for a given token, nor use the stand alone service catalog as a
means of authorization.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This will enable end users to obtain a service catalog using v3 tokens that
were generated without a service catalog.

Both ``python-keystoneclient`` and ``keystonemiddleware.auth_token`` should add
support for the new call without any user intervention.

Performance Impact
------------------

This change will potentially increase network chattiness in favor of smaller
tokens.

``keystonemiddleware.auth_token`` might want to cache the service catalog for a
token to avoid repeating catalog requests back to Keystone.

Tokens will be generated slightly faster, as a resulting of eliminating calls
to the catalog driver.

Tokens may be substantially reduced in size, effectively reducing HTTP packet
sizes for all services (especially beneficial to those that do not utilize the
service catalog).

Other Deployer Impact
---------------------

``keystonemiddleware.auth_token`` might benefit (in terms of performance) from
a new configuration option to indicate that the protected service does not
require a catalog.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dolph

Other contributors:
  None

Work Items
----------

- add new API call to ``openstack/identity-api``
- add service-side implementation to ``openstack/keystone``
- add support to ``python-keystoneclient`` for retrieving a service catalog
  when faced with a token that lacks one
- add support to ``keystonemiddleware.auth_token`` to retrieve a service
  catalog when faced with a token that lacks one (and the underlying service
  expects a catalog)

Dependencies
============

The addition of authentication specific routes to the ``/auth`` path is part of
the ``auth-specific-data`` blueprint. However there is no dependency between
the functionality of the different blueprints.

Documentation Impact
====================

The proposed API will need to be documented on ``openstack/openstack-api-site``
as part of Identity API v3.

References
==========

None

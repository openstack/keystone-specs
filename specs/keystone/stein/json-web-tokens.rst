..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================================
Add JSON Web Tokens as a Non-persistent Token Provider
======================================================

`bp json-web-tokens <https://blueprints.launchpad.net/keystone/+spec/json-web-tokens>`_


JSON Web Token is a type of non-persistent bearer token similar to the fernet
tokens we use today. JWT is an `open standard`_ with `actively maintained
libraries`_.

.. _`open standard`: https://tools.ietf.org/html/rfc7519
.. _`actively maintained libraries`: https://jwt.io/#libraries

Problem Description
===================

We currently support one token format called fernet. The fernet token format is
a non-persistent format based on a spec by Heroku and was made the default
token format for keystone.

However, the fernet format has its own problems that make it non-ideal. The
`fernet spec`_ is largely abandoned, making it hard to `get changes into it`_
and thereby into the `cryptography`_ implementation of it. Moreover, the fernet
spec is not recognized by any standards body and therefore not as closely
audited as an IETF standard, making it more susceptible to zero-day
vulnerabilities. Addressing these vulnerabilities falls solely on the
OpenStack, specifically the keystone, community.

It would be nice to offer a new type of token that is backed by a widely used
standard. This also increases the chances of interoperability between the
OpenStack ecosystem and other communities that support JWT.

.. _`get changes into it`: https://github.com/fernet/spec/pull/13

Use Cases
---------

* As a operator, I want to use a non-persistent token provider that isn't
  coupled to a symmetric encryption or signing implementation. An
  implementation built on asymmetric signing or encryption allows me to
  distribute public keys from one node to another instead of syncing a
  repository of symmetric keys. This makes it easier to deploy keystone nodes
  with read-only capabilities strictly used for token validation. The
  specific usecase for this allows me to deploy read-only regions keeping token
  validation within the region, while having tokens issued from a central
  identity management system in a separate region.

* As an operator, I want to be have a token provider to fall back on in the
  event there is a security vulnerability in the `fernet spec`_ or the
  `cryptography`_ implementation consumed by keystone.

* As a user, I want to be able to authenticate for a token that I can use with
  other pieces of software outside of the OpenStack ecosystem that prove my
  identity.

.. _`fernet spec`: https://github.com/fernet/spec/blob/master/Spec.md
.. _`cryptography`: https://github.com/pyca/cryptography

Proposed Change
===============

Create a new non-persistent keystone token backend based on the `JSON Web Token
standard`_. These will behave in much the same way as our current fernet tokens
do.

The token will be a signed JWT (`JWS`_) containing the authentication payload.
Note that signed tokens are web safe and integrity verified, but token payload
is not opaque to its holder. It is possible to decode a token and inspect the
payload with `JWS`_ tokens. Using nested `JWE`_ tokens are the JSON Web Token
equivalent to Fernet tokens and they are encrypted and signed.

This implementation reserves the right to change, modify, or remove items in
the payload at any point in time and for any reason. Decoding and relying on
attributes within the payload is not a supported API, nor should it be assumed
as one. Users should use formal APIs to request information from keystone. The
development team will not prevent or stall changes to token payloads, which are
internal implementation details of the token provider, due to users relying on
those attributes in ways they shouldn't. Likewise, deployment that consider
information in the token payload sensitive should rely on Fernet to prevent
that information from being exposed to users.

Similar to the Fernet, JWTs will require a key repository be set up to use for
signing tokens. A new ``keystone-manage`` command will be added to handle
secret generation and rotation which will likely re-use much of the utilities
in the ``fernet_setup`` and ``fernet_rotate`` commands. The recommended
algorithm to be used is ``ES256``.  Keystone should not expose the ability to
end users to ask for a specific JWS algorithm. Support should be limited to
only supported or trusted algorithms that the end user cannot specify. JWS
tokens will be integrity verified with a private key and validated using a
corresponding public key. Since the ``ES256`` implementation only uses signing
(as opposed to signed encrypted payloads), this adheres to slightly better
security practices over fernet because private keys never have to be synced
across keystone API nodes. Only public keys need to be transferred to other
keystone API servers to validate tokens across a cluster.

The payload of the JWS will use the following `registered claims`_:

* the `sub` claim will be a **required** string containing the ID of the user
  who authenticated for the token
* the `exp` claim will be a **required** numeric value for token expiration
* the `iat` claim will be a **required** numeric value for the time a token was
  issued

The following private claims will be used to relay additional required
information and will be prefixed with `openstack` to avoid collisions with
future upstream claims:

* `openstack_methods` will be a **required** claim denoting the authentication
  methods used to obtain the token
* `openstack_audit_ids` will be a **required** claim containing the audit
  information associated to a token
* `openstack_system` will be an **optional** claim only present in
  system-scoped tokens
* `openstack_domain_id` will be an **optional** claim only present in
  domain-scoped tokens
* `openstack_project_id` will be an **optional** claim only present in
  project-scoped tokens
* `openstack_trust_id` will be an **optional** claim only present in
  trust-scoped tokens
* `openstack_app_cred_id` will be an **optional** claim only present in
  application credential tokens
* `openstack_group_ids` will be an **optional** claim only present in federated
  tokens to carry an ephemeral user's group assignments
* `openstack_idp_id` will be an **optional** claim only present in federated
  tokens to carry the ID of a user's identity provider
* `openstack_protocol_id` will be an **optional** claim only present in
  federated tokens to denote the protocol used by a federated user to
  authenticate
* `openstack_access_token` will be an **optional** claim only present in OAuth
  tokens

The `PyJWT library`_ is already present in the requirements repository and
would be a convenient choice to use for this implementation. Both the `PyJWT
library`_ and the `JWCrypto library`_ implement support for JWS. Since the
implementation detailed in this specification is unique to ``ES256``, a library
that supports JWE isn't necessary. If supporting another encrypted token type,
like fernet, is a requirement in the future, then finding or contributing JWE
support to the consuming library would be necessary.

Users will request and present tokens in exactly the same way they currently do
with Fernet tokens. There is no need to add or change any APIs.

.. _`JSON Web Token standard`: https://tools.ietf.org/html/rfc7519
.. _`JWS`: https://tools.ietf.org/html/rfc7515
.. _`JWE`: https://tools.ietf.org/html/rfc7516
.. _`registered claims`: https://tools.ietf.org/html/rfc7519#section-4.1
.. _`Python libraries`: https://jwt.io/#libraries
.. _`PyJWT library`: https://pyjwt.readthedocs.io/en/latest/
.. _`does not yet support JWE`: https://github.com/jpadilla/pyjwt/issues/143
.. _`JWCrypto library`: http://jwcrypto.readthedocs.org/

Key Setup & Rotation
--------------------

Much like the Fernet implementation, a JWT provider will require a key rotation
strategy. Since ``ES256`` relies on asymmetric signing, the suggested rotation
strategy will be slightly different than what is known with Fernet.

The Fernet implementation requires the usage of a staged key, which is just a
key with a special name, in order to ensure tokens can be validated during the
rotation process. This won't be required with JWT and the following steps
should be sufficient to perform key rotation without token invalidation due to
missing signing keys. Assume the following steps are being performed on three
different API servers, named `K1`, `K2`, and  `K3`, that need to validate
tokens issued by each other.

1. A key pair is created for each API server. `K1.private`, `K1.pub` for
   `K1`, `K2.private`, `K2.pub` for `K2`, and `K3.private`, `K3.pub` for `K3`.
2. A copy of each public key is transferred to each API server. `K1`, `K2`, and
   `K3` all have copies of `K1.pub`, `K2.pub`, and `K3.pub`.

At this point, tokens issued from any API server can be validated anywhere. In
the event a single API server needs to rotate keypairs:

1. A new key pair is created for `K1` called `K1-new.private` and `K1-new.pub`.
   `K1` is configured to start signing tokens with both `K1.private` and
   `K1-new.private.`
2. `K1-new.pub` is copied to the public key repository of each API server. So
   long as `K2` and `K3` have either `K1.pub` or `K1-new.pub` they can validate
   tokens issued by `K1`.
3. After `K2` and `K3` have been updated with copies of `K1-new.pub`,
   `K1.private` can be removed from `K1` and `K1.pub` can be removed from `K2`
   and `K3`. Tokens that were signed with only `K1.private` are unable to be
   verified and `K1.pub` should only be removed after those tokens have expired
   anyway.

Traditional asymmetric keys can be revoked using revocation lists. At this time
we are not going to support a revocation list implementation for JWT key pairs.
The operator has the ability to sync public keys accordingly when they rotate
new keys in and out. Keystone will only use the public keys on disk to validate
tokens. Is could change in the future, but for now it keeps the key rotation
and key utilities with keystone simpler.

Crypto-Agility & Future Work
----------------------------

This specification is targeting a single algorithm for the initial JWT
implementation. If and when keystone decides to expand the implementation to
include additional algorithms, we should allow for flexibility between
configured algorithms, which will make it easier for operators to switch from
one algorithm to another if they need to.

For example, the validation process using a JWT token provider might support
validating multiple blessed algorithms, allowing multiple tokens signed with
different algorithms to be validated without require configuration changes
except on the signing node.

Alternatives
------------

Recently, there have been various efforts that help solve authenticated
encryption. One of these efforts was sparked by a `concern`_ with JWT, namely
the `JOSE`_ header. The issue detailed in the report was specific to users
being able to specify algorithms and exploit a validation weakness in various
JWT libraries. All python libraries have been patched, but keystone should
specifically rely on validating algorithm usage and never assuming algorithms
to be supplied by end users. Please see the full `report`_ for details on the
vulnerability and why we are going to strictly validate this information.

There is a proof-of-concept implementation for Platform Agnostic Security
Tokens, or `PASETO`_ that takes a more strict stance on algorithm validation
and the intended audience of the token. The strict stance of `versioned
protocols` with `PASETO`_ is certainly advantageous, but the implementation and
idea are still in the incipient stage. It's certainly worth noting that we
should keep out eye on this development and re-evaluate it if, or when, it gets
more adoption.

For now, if keystone supplies strict algorithm validation to the JWT
implementation, we should be able to offer a comparable backup option to
fernet.

.. _`concern`: https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
.. _`report`: https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
.. _`JOSE`: https://tools.ietf.org/html/rfc7519#section-5
.. _`PASETO`: https://github.com/paragonie/paseto

Security Impact
---------------

Since JWT is a widely used web standard, this will have a net positive impact
on security. The implementation will use asymmetric signing, reducing risk of
having to replicate or transfer private keys from one host to another. Since
the token payloads are signed, data within the token will be readable to anyone
who has the token. The token can only be validated using the corresponding
public key of the private key used to sign the token originally.  These will
still be bearer tokens and so interception of one must still be guarded
against.

Known Vulnerabilities
~~~~~~~~~~~~~~~~~~~~~

There is a documented `vulnerability`_ that affected several JWT libraries,
including one library written in Python.

In most cases, JSON Web Tokens will have a header, payload, and signature where
each section is delimited by a period (``.``). The header contains an important
piece of information, which is how the token's integrity is protected. This is
stored as the ``alg`` attribute of the header. The library verifying the token
uses the algorithm specified in the header to perform an integrity check and
compares its results to the signature portion of the token.

Security concerns have been documented and raised that describe the issues with
allowing clients to dictate algorithms used for token verification. This is a
concern specifically with applications that support asymmetric and symmetric
signing. An attacker could effectively bypass the verification check of a
token by using a published, or known, public key to generate a JWT with a
symmetric signing algorithm.

This would be applicable if keystone supported signed tokens and encrypted
tokens with the same token provider implementation. This vulnerability has been
addressed across various libraries after its discovery, but keystone should be
aware of the overall technique that lead to it in the first place. We can
mitigate this type of vulnerability in keystone by:

* Ensuring keystone doesn't blindly allow end users to specify which algorithm
  is used to verify the integrity of a token (e.g., only implementing support
  for ``ES256``)
* Ensure the ``alg`` supplied in the token header is only ever populated by
  keystone
* Ensure keystone only issues tokens of a single encryption or signing strategy
  (e.g., not allowing users to get signed token and encrypted tokens from the
  same server, thus mixing asymmetric and symmetric key usage at runtime)

Specifics about the `vulnerability`_ can be found in the report.

.. _`vulnerability`: https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/

Notifications Impact
--------------------

Notifications for JWTs will behave in the same way that they do for fernet
tokens, including for revocation events.

Other End User Impact
---------------------

This will have no end user impact. They will request and use JWTs in exactly
the same way that they currently use fernet tokens.

Performance Impact
------------------

It will be worth investigating performance differences between token providers
that use asymmetric signing (JWT) and symmetric encryption (fernet). These
difference, if significant, should be published in documentation as it might be
useful for operators when choosing a token provider.

Other Deployer Impact
---------------------

This is an optional, opt-in feature that will not be the default, so deployers
will not be affected unless they choose to use JWT. In that case, deployers
will need to set up a key repository before using JWTs. The key repository will
contain asymmetric key pairs rather than just secret keys. The deployer will
need to take care to sync and rotate keys the way they do with fernet tokens.

Developer Impact
----------------

The new token type will reuse much of the work already done for fernet tokens
and will follow similar code paths, so this will be relatively easy to
maintain.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gage Hugo (gagehugo)
  Lance Bragstad (lbragstad)
  XiYuan Wang (wxy)

Work Items
----------

* Refactor the fernet utilities modules to be generic enough to work with JWT
  or inheritable
* Add a ``keystone-manage`` command to set up and rotate JWT signing keys
* Generalize the ``TokenFormatter`` class to support JWT
* Refactor the fernet token provider module to be inheritable or generic
* Add a keystone doctor command to validate the setup in the same way that
  fernet is validated


Dependencies
============

There are three different libraries we can use to implement this functionality.

1. `PyJWT`_

   This library only supports token signing, or JWS. It does not support JWE,
   or authenticated encryption, yet. A minimum version of **1.0.1** is
   `required`_, but this library is already included in OpenStack global
   requirements repository.

2. `python-jose`_

   This library only supports token signing, or JWS. It does not support JWE,
   or authenticated encryption, yet. This library is not included in OpenStack
   global requirements.

3. `JWCrypto`_

   This library supports both JWS and JWE, but it is not included in OpenStack
   global requirements.

3. `Authlib`_

   This library supports both JWS and JWE, but its licensing is incompatible
   with OpenStack as it is AGPL.

Given the fact that the initial implementation of JWT is not going to rely on
nested JWT tokens or encrypted payloads, it's safe to assume that signing
support will be sufficient. The PyJWT library is already included in global
requirements and we don't have a case to not use that specific library, which
is compatible with OpenStack licensing.

.. _`PyJWT`: https://pyjwt.readthedocs.io/en/latest/
.. _`required`: https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
.. _`python-jose`: https://python-jose.readthedocs.io/en/latest/
.. _`JWCrypto`: http://jwcrypto.readthedocs.io/en/latest/
.. _`Authlib`: https://docs.authlib.org/en/latest/

Documentation Impact
====================

The new ``[token]/provider`` configuration option will need to be documented,
as will the new ``keystone-manage`` commands.


References
==========

* `JSON Web Token RFC <https://tools.ietf.org/html/rfc7519>`_
* `JSON Web Token light introduction <https://jwt.io/introduction/>`_
* `History of cryptography's adoption of fernet <https://github.com/pyca/cryptography/issues/2900>`_

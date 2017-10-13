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

We currently support two types of tokens. The UUID persistent token format is
the original format. The fernet token format is a non-persistent format based
on a spec by Heroku and was made the default token format for keystone.

The UUID token format suffers a number of performance and maintainability
problems. It is already deprecated and we intend to eventually remove it, in
favor of non-persistent token formats.

However, the fernet format has its own problems that make it non-ideal. The
`fernet spec`_ is largely abandoned, making it hard to `get changes into it`_
and thereby into the `cryptography` implementation of it. Moreover, the fernet
spec is not recognized by any standards body and therefore not as closely
audited as an IETF standard, making it more susceptable to zero-day
vulnerabilities. Addressing these vulnerabilities falls solely on the
OpenStack, specifically the keystone, community.

It would be nice to offer a new type of token that is backed by a widely used
standard before we totally remove support for UUID tokens. This also increases
interoperability between the OpenStack ecosystem and other communities that
support JWT.

.. _`fernet spec`: https://github.com/fernet/spec/blob/master/Spec.md
.. _`get changes into it`: https://github.com/fernet/spec/pull/13


Proposed Change
===============

Create a new non-persistent keystone token backend based on the `JSON Web Token
standard`_. These will behave in much the same way as our current fernet tokens
do.

The token will be a nested JWT which is a signed JWT (`JWS`_) as the payload of
an encrypted JWT (`JWE`_), meaning the token data is signed and then encrypted.
This is the order specifically recommended in the JWT spec due to its use of
Authenticated Encryption which eliminates the need to sign after encryption.

Similar to the fernet, JWTs will require a key repository be set up to use for
signing tokens. A new ``keystone-manage`` command will be added to handle
secret generation and rotation which will likely re-use much of the utilities
in the fernet_setup and fernet_rotate commands. JWE can use symmetric keys for
encryption and signing the way fernet does but it is more typical to use an
asymmetric key pair to encrypt the Content Encryption Key (which is then used
as a symmetric key to encrypt the payload, but this is an implementation detail
that will be handled by the chosen library), so the JWT key repository should
have asymmetric key pairs which we can use for encryption and signing. The
specific algorithms used will depend on the support for them in the chosen
library.

The payload of the JWS will use the following `registered claims`_:

* the "sub" claim for the subject, i.e. the user. This claim will include the
  same data that fernet tokens do, such as user information and scope.
* the "exp" claim for expiry
* the "iat" claim for "issued at", analogous to the timestamp field in the
  fernet token.

The `PyJWT library`_ is already present in the requirements repository and so
would be a convenient choice to use for this implementation, but it
unfortunately `does not yet support JWE`_. We will need to evaluate the
`JWCrypto library`_ which does support encryption or consider contributing the
feature to PyJWT.

Users will request and present tokens in exactly the same way they currently do
with either UUID or Fernet tokens. There is no need to add or change any APIs.

.. _`JSON Web Token standard`: https://tools.ietf.org/html/rfc7519
.. _`JWS`: https://tools.ietf.org/html/rfc7515
.. _`JWE`: https://tools.ietf.org/html/rfc7516
.. _`registered claims`: https://tools.ietf.org/html/rfc7519#section-4.1
.. _`Python libraries`: https://jwt.io/#libraries
.. _`PyJWT library`: https://pyjwt.readthedocs.io/en/latest/
.. _`does not yet support JWE`: https://github.com/jpadilla/pyjwt/issues/143
.. _`JWCrypto library`: http://jwcrypto.readthedocs.org/

Alternatives
------------

Not applicable, this is an additive change. The alternative is not to add it or
to find a different token format.

Security Impact
---------------

Since JWT is a widely used web standard, this will have a net positive impact
on security. Choosing to use JWE, an optional feature of the JWT spec, will
ensure that the data within the token is at least as secure as it is in fernet
tokens. These will still be bearer tokens and so interception of one must still
be guarded against.

Notifications Impact
--------------------

Notifications for JWTs will behave in the same way that they do for fernet
tokens, including for revocation events.

Other End User Impact
---------------------

This will have no end user impact. They will request and use JWTs in exactly
the same way that they currently use fernet or UUID tokens.

Performance Impact
------------------

Performance will be on-par with fernet tokens.

Other Deployer Impact
---------------------

This is an optional, opt-in feature that will not be the default, so deployers
will not be affected unless they choose to deploy this feature. In that case,
deployers will need to set up a key repository before using JWTs. The key
repository will contain asymmetric key pairs rather than just secret keys. The
deployer will need to take care to sync and rotate keys the way they do with
fernet tokens.

Developer Impact
----------------

The new token type will reuse much of the work already done for fernet tokens
and will follow similar code paths, so this will be relatively easy to
maintain.

Implementation
==============

Assignee(s)
-----------

TBD: Please update this section when this spec is reproposed to a release.

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

* Decide on an implementation library
* Refactor the fernet utilities modules to be generic enough to work with JWT
  or inheritable
* Add a ``keystone-manage`` command to set up and rotate JWT signing/encryption
  keys
* Generalize the ``TokenFormatter`` class to support JWT
* Refactor the fernet token provider module to be inheritable or generic
* Add a keystone doctor command to validate the setup in the same way that
  fernet is validated


Dependencies
============

* A JWT library to be decided on


Documentation Impact
====================

The new ``[token]/provider`` configuration option will need to be documented,
as will the new ``keystone-manage`` commands.


References
==========

* `JSON Web Token RFC <https://tools.ietf.org/html/rfc7519>`_
* `JSON Web Token light introduction <https://jwt.io/introduction/>`_
* `History of cryptography's adoption of fernet <https://github.com/pyca/cryptography/issues/2900>`_

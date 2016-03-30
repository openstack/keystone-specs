..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Support TOTP Authentication
===========================

`bp totp-auth <https://blueprints.launchpad.net/keystone/+spec/totp-auth>`_

Support Time-Based One-Time Password (TOTP) Authentication as a distinct
authentication mechanism. From `Wikipedia
<https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm>`_:

    Time-based One-time Password Algorithm (TOTP) is an algorithm that computes
    a one-time password from a shared secret key and the current time. It has
    been adopted as Internet Engineering Task Force standard RFC 6238, is the
    cornerstone of Initiative For Open Authentication (OATH), and is used in a
    number of two-factor authentication systems.

The algorithm proposed is used by Google to provide a "possession factor" as
part of their approach to multi-factor authentication (Gmail, Google
Authenticator). The same would apply to Keystone.

Problem Description
===================

Keystone does not have formal support for multi-factor authentication, nor do
we formally support additional authentication factors beyond "knowledge
factors" (something you know, such as a password). In order to provide stronger
authentication, we must first support a second factor, such as a "possession
factor" (something you have, such as a one-time password generating device) or
an "inherence factor" (something associated with the user, such as a
fingerprint). This spec proposes to provide a "possession factor" as a
prerequisite to true multi-factor authentication.

In order to support multiple factors, a deployer will need to continue to
support both the existing password mechanism (for example) in addition to the
TOTP mechanism. In order to be able to distinguish which to use in a given
authentication request, TOTP needs to be a distinct authentication plugin.

Proposed Change
===============

Add a ``totp``` auth method.

Example request that uses only the ``totp`` authentication plugin (this is not
particularly useful by itself in the real world, but illustrates that the
``totp`` plugin is not treated any differently when compared to existing
plugins)::

  {
      "auth": {
          "identity": {
              "methods": [
                  "totp"
              ],
              "totp": {
                  "user": {
                      "id": "b95b78b67fa045b38104c12fb2729cd0",
                      "passcode": "012345"
                  }
              }
          },
          "scope": {
              "project": {
                  "id": "9c1c4b2657a04f7fbd46237835a43f59"
              }
          }
      }
  }

Example request that uses both ``totp`` and the existing ``password`` plugin::

  {
      "auth": {
          "identity": {
              "methods": [
                  "password", "totp"
              ],
              "password": {
                  "user": {
                      "id": "b95b78b67fa045b38104c12fb2729cd0",
                      "password": "password"
                  }
              },
              "totp": {
                  "user": {
                      "id": "b95b78b67fa045b38104c12fb2729cd0",
                      "passcode": "012345"
                  }
              }
          },
          "scope": {
              "project": {
                  "id": "9c1c4b2657a04f7fbd46237835a43f59"
              }
          }
      }
  }

Example request that uses both the ``token`` and the ``totp`` authentication
method::

  {
      "auth": {
          "identity": {
              "methods": [
                  "token", "totp"
              ],
              "token": {
                  "id": "'$OS_TOKEN'",
              },
              "totp": {
                  "user": {
                      "id": "b95b78b67fa045b38104c12fb2729cd0",
                      "passcode": "012345"
                  }
              }
          },
          "scope": {
              "project": {
                  "id": "9c1c4b2657a04f7fbd46237835a43f59"
              }
          }
      }
  }


The ``totp`` plugin can be swapped in by updating Keystone configuration. There
will be multiple implementations of the TOTP plugin based on the backend used
to validate.

The auth plugin will require a user to identify themselves and provide a
client-generated one time password. The plugin will then verify that the one
time password is valid for the user.

* Keystone will validate any enabled credentials with credential type "totp"
  for the given user and use them to validate the HMAC.

* If one matches, the authentication is considered successful.

* If none match, authentication is considered unsuccessful.

* The initial implementation will not support a time sync compensation
  mechansim since we are expecting the client-server time drift to be fairly
  minimal.

Alternatives
------------

None

Security Impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* A user's secret key to generate one-time passwords will need to be stored and
  encrypted. Later on this storage could be done in Barbican. We could leverage
  Fernet to encrypt these seeds which would mean they are AES 256 encrypted.

* The proposed algorithms involve the use of cryptography (HMAC) for TOTP
  calculation.


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

* The OTP driver should be specified in the Keystone configuration file.

* This change would add a new, optional authentication plugin for TOTP that is
  not enabled by default.

  Example pseudo-configuration in ``keystone.conf``::

    [auth]
    methods = password, totp
    totp = pkg.path.mfa.Totp

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  werner.mendizabal (Werner Mendizabal <nonameentername@gmail.com>)

Work Items
----------

* Add server-side support in the form of a new authentication method in
  keystone.

* Add client side support in the form of a new auth plugin to keystoneauth.

* Document the API and auth plugin examples using keystoneauth.

* Add documentation that explains how to enable and configure the plugin in
  keystone.

Dependencies
============

None

Documentation Impact
====================

None

References
==========

* `Time-based One-time Password Algorithm (Wikipedia)
  <https://en.wikipedia.org/wiki/Time-based_One-time_Password_Algorithm>`_

* `Google Authenticator
  <https://code.google.com/p/google-authenticator>`_

* `TOTP (IETF)
  <https://tools.ietf.org/html/rfc6238>`_

* `Knowledge factor (Wikipedia)
  <http://en.wikipedia.org/wiki/Multi-factor_authentication#Knowledge_factors>`_

* `Possession factor (Wikipedia)
  <http://en.wikipedia.org/wiki/Multi-factor_authentication#Possession_factors>`_

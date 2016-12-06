..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Per-User Auth Plugin Requirements
=================================

`bp per-user-auth-plugin-reqs <https://blueprints.launchpad.net/keystone/+spec/per-user-auth-plugin-reqs>`_


Keystone should be able to support the ability to require (on a per-user basis)
the authentication plugins (the ones supported in the keystone server) that
are required to authenticate.

Problem Description
===================

In many services (web or otherwise) it is possible to require certain users to
utilize different authentication methods. This is currently not the case in
Keystone. Keystone has the ability to support multiple authentication types
(such as password, totp, x509, etc). Keystone cannot require that a user
(or even a group of users) must utilize a specific (or a combination of) plugin
for authentication; the use of any of the plugins (in isolation) is sufficient
to provide authentication.

Proposed Change
===============

Keystone will support the ability to require (on a per-user basis) certain
authentication plugins as required to provide authentication. This means that
a user can be required to use a grouping of auth plugins (e.g. must use either
"password AND totp" OR "x509" OR "password AND one-time-backup-code"). If a
user does not have a rule (or set of rules, in the case of logical ORs)
configured, the current default of "any single auth plugin configured in
keystone.conf" will continue to work.

The rules will be represented in JSON:

.. code-block:: json

  "required_auth_plugins": [
    ["password", "totp"],
    ["x509"],
    ["password", "one-time-backup"]
  ]

The value of these required plugins will be added as a column to the user in
the database schema as a ``blob`` type (serialized as JSON).

In the case that a required plugin is configured but is not enabled in the
Keystone.conf, that part of the rule is simply ignored (e.g. "password AND
totp" is the rule, but keystone.conf only has "password" enabled, "totp" will
simply be ignored as a requirement for authentication purposes for any/all
users).

If there is only one plugin "required" for a user and that plugin is
disabled keystone will fall back to the default behavior of "any one plugin
configured is sufficient".

The authentication request will conform to the current keystone standard for
multiple auth-plugin forms:

.. code-block:: json

    {
        "auth": {
            "identity": {
                "methods": [
                    "password",
                    "totp"
                ],
                "password": {
                    "user": {
                        "id": "0ca8f6",
                        "password": "secretsecret"
                    }
                },
                "totp": {
                    "user": {
                        "id": "0ca8f6",
                        "passcode": "011011"
                    }
                }
            },
            "scope": {
                "domain": {
                    "id": "1789d1"
                }
            }
        }
    }

If insufficient auth plugins are supplied a ``HTTP 401`` with a JSON response
indicating insufficient auth parameters will be returned to the requestor. This
response will be the same regardless of correct-ness of any of the
values passed to the authentication plugins. The plugins required for a given
request will be compared to the plugin data supplied will be compared prior to
validating the data supplied to the plugins.

Longer term, more fluid auth work-flows such as supplying user, then password,
then OTP (across multiple pages) may be developed utilizing temporary auth
tokens that allow stepping through the workflow. However, these have potential
to leak secure information (existence of user, validation of password, etc) if
improperly implemented/handled.

A self-service API (similar to password change) will be required to enable
the enforcement rules for a given user. `update-user` could be used by an
admin to make the change directly.

Alternatives
------------

Utilizing an external IDP via Federated Authentication can add in MFA style
support if the IDP (such as "FreeIPA") supports the concept of MFA natively.

It is possible to add many more forms of auth plugins which have business
logic encoded in them. An example would be a password+totp plugin that would
strip the TOTP from the end of the password if the user has a credential with
the TOTP type, essentially making TOTP+Password required, but allowing the
current work-flow that is in use today. The downside is that the auth-plugins
would need to be extremely logic heavy as more auth methods are created instead
of allowing for a rule-set to manage which auth-plugins are required.

Security Impact
---------------

TBD

Notifications Impact
--------------------

NONE

Other End User Impact
---------------------

End users would need to auth with the required auth-plugins specified in the
rules for that user if rules are enabled.

Performance Impact
------------------

Authentication may see a slight slowdown as more than one auth plugin
will need to be processed. Overall performance should remain about the same
as today.

Other Deployer Impact
---------------------

Deployers wishing to enforce use of multiple auth-types will need to
create the users with the new rules (and/or update current users). If the
deployer does not want users to update the auth-plugin requirements, policy
will need to be updated to deny access to the new self-service
auth-plugin-requirements API.

Developer Impact
----------------

No significant impact.


Implementation
==============

Assignee(s)
-----------

Primary assignee(s):
    Morgan Fainberg <mdrnstm>
    Adrian Turjak <adriant-y>

Other contributors:
    N/A

Work Items
----------

* Implement database migration to add new column for users

* Support requiring the auth-types specified in the new "required_auth_plugins"
  attribute when authenticating.

* Implement self-service API for updating required auth plugins

* Write Documentation (API-REF) about the updated forms of authentication
  and new self-service API.

* Add support to keystoneclient for self-service API

* Add support to keystoneauth for better handling insufficient auth-types
  supplied.

* Ensure keystoneauth supports multiple auth plugins at once.

* Work with Horizon and Openstackclient Teams to ensure support for new
  multiple-auth-types are handled with a good UI/UX.


Dependencies
============

No External Dependencies.


Documentation Impact
====================

Documentation for new APIs and new auth functionality will be required.

References
==========

No external references

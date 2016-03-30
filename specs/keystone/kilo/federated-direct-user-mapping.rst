..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Direct users mapping for federated authentication
=================================================

`bp federated-direct-user-mapping
<https://blueprints.launchpad.net/keystone/+spec/federated-direct-user-mapping>`_

Allow for specifying mapping rules where effective user exists in the
backend. Otherwise treat as ephemeral users, members of service domain called
``Federated``.

Problem Description
===================

Today mapping engine assumes all effective users are ephemeral and
intentionally does not do any database lookups.
Many cloud providers want to use cloud federation so the authentication is
handled by the first class Identity Provider, but the user exists in the
Identity backend.

This problem is manifested in many ways, firstly when an ephemeral user obtains
a token, the user section of the token does not have a 'domain' portion. This,
breaks the API contract we established when obtaining tokens. Currently,
consuming projects work around this issue by checking to see if 'OS-FEDERATION'
is present in the token.

Proposed Change
===============

Part of the change is adding a concept of a default federated domain. That is,
all the ephemeral users (not present in the backend) will be stored in a
default federated domain. The domain's name will be immutable and will be named
``Federated``.

Even though there are many users from many IdPs being placed in the default
``Federated`` domain, there is no collision between user ids and names, since
the users do not exist in Keystone, and domains are a Keystone concept.
The user's token will provide information about the Identity Provider that was
used to authenticate.

Mapping rules should be changed - cloud administrators should be able to
specify the domain the user belongs to. If such an attribute is not specified
explicitly, this will mean the user is meant to be ephemeral and automatically
belongs to federated domain. If the domain is not present in the backend,
server will respond with appropriate error code.  If the  domain attribute is
specified and is present in the backend, server will do the user lookup
and respond with an unscoped token. The token should be later scoped to a
project or domain, depending on roles assigned to the user. Specifying
additional groups in the mapping rules will not take any effect on roles the
user has assigned to him. Domain can be identified either by ``id`` or
``name``.
Example of ``local`` rule specifying a domain in the ``user`` object:

::

    "local": [
        {
            "user": {
                "id": "{0}",
                "domain": {
                    "id": "12de34"
                }
            }
        }
    ]


In case the effective user is ephemeral, an unscoped federated token will be
issued:

::

    {
        "token": {
            "methods": [
                "saml2"
            ],
            "user": {
                "id": "username%40example.com",
                "domain": {
                    "name": "Federated"
                },
                "name": "username@example.com",
                "OS-FEDERATION": {
                    "identity_provider": "ACME",
                    "protocol": "SAML",
                    "groups": [
                        {"id": "abc123"},
                        {"id": "bcd234"}
                    ]
                }
            }
        }
    }


The proper way to distinguish whether the user is ephemeral or not is by
checking by membership in the federated domain. If no domain information is
provided, then it is assumed that the user is ephemeral and will use the
default federated domain.

Proposed change also solves the issue of an inconsistent token being returned,
(where the user didn't have a domain) and there should be less special handling
needed by clients/middleware.

Alternatives
------------

None.

Security Impact
---------------

Leveraging authentication to a first class Identity Provider can increase
overall security.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

Federated workflow includes more HTTP calls to provide a token to an end user.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Marek Denis (marek-denis)

Other assignees:
    Steve Martinelli (stevemar)
    Guang Yee (gyee)

Work Items
----------

* Add logic for service domain ``Federated``.
* Mapping engine to properly handle ``domain`` keyword in the ``user`` object.
* Adjust auth plugins to distinguish between ephemeral user authentication and
  existing user mapping.
* In case of ephemeral user change the format of the unscoped tokens

Dependencies
============

None.

Documentation Impact
====================

All the changes must be reflected in the documentation.

References
==========
Add a domain to federated users - https://review.openstack.org/#/c/110858/

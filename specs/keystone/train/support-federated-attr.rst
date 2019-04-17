..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Extend user API to support federated attributes
===============================================

`bp support-federated-attr <https://blueprints.launchpad.net/keystone/+spec/support-federated-attr>`_

Extend the user API to support federated attributes so that we treat federated
users like any other Keystone user.

Problem Description
===================

With the addition of shadow users [1], federated users are no longer ephemeral
and are like any other Keystone user. Thus, federated users have a record in
the user table and can receive concrete role assignments. However, in order to
assign a federated user a role, you would need the local user ID. Likewise, you
would need the user ID to do delegation and create a trust. However, we don't
have an API method that supports querying federated users in order to get the
local user ID.

Another question we have been dealing with, is how to provision authorization
to federated users, since a federated user does not exist in Keystone until
they authenticate for the first time. In addition, at the Barcelona Summit, we
discussed a need for operators to be able to easily change and rollback
provisioning.

Proposed Change
===============

To solve this problem, we could simply extend the user API to support federated
attributes. For example, creating a federated user:

.. code:: javascript

    POST /v3/users
    {
        "user": {
            "default_project_id": "263fd9",
            "domain_id": "1789d1",
            "enabled": true,
            "name": "James Doe",
            "federated": [
                {
                    "idp_id": "1789d1",
                    "protocols": [
                        {
                            "protocol_id": "saml2",
                            "unique_id": "jdoe"
                        }
                    ]
                }
            ]
        }
    }

Response example:

.. code:: javascript

    {
        "user": {
            "domain_id": "1789d1",
            "id": "9fe1d3",
            "name": "James Doe",
            "enabled": true,
            "links": {
                "self": "https://example.com/identity/v3/users/9fe1d3"
            }
            "federated": [
                {
                    "idp_id": "1789d1",
                    "protocols": [
                        {
                            "protocol_id": "saml2",
                            "unique_id": "jdoe"
                        }
                    ]
                }
            ]
        }
    }

Creating a federated object for an existing user:

.. code:: javascript

    PUT /v3/users/{id}
    {
        "federated": {
            "idp_id": "1789d1",
            "protocols": [
                {
                    "protocol_id": "saml2",
                    "unique_id": "jdoe"
                }
            ]
        }
    }

Response example:

.. code:: javascript

    {
        "federated": {
            "idp_id": "1789d1",
            "protocols": [
                {
                    "protocol_id": "saml2",
                    "unique_id": "jdoe"
                }
            ]
            "links": {
                "self": "http://.../v3/users/{id}/federated/{id}"
            }
        }
    }

Likewise, you could query for a specific federated user by querying the
federated unique_id:

.. code:: javascript

    GET /v3/users/?unique_id={unique_id}

If the unique_id was not unique across the organization, the request would need
to include additional parameters in order to return the specific user.

Extending the API gives operators a way to get the local user ID for federated
users in order to do provisioning. And the user API is a natural place for
these operations, as a federated user is in fact a user. The federated
attributes live within the user data model in the sql backend.

This could also go hand in hand with shadow mapping [2], allowing operators to
provision in mass, as well as having the flexibility to fully utilize the API
for managing federated identity. The bottom line, lets treat federated users
like any other Keystone user.

Alternatives
------------

Continue with shadow mapping [3] as the only option for providing federated
user provisioning.

We could extend OS-FEDERATION to support federated user operations.

Security Impact
---------------

None

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

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

- Kristi Nikolla

Other contributors:

- Ron De Rose

Work Items
----------

1. Extend user create API to support federated attributes.

2. Extend other user API methods as needed.

3. Update docs

Dependencies
============

None

Documentation Impact
====================

We would need to update the user API docs, as well as the federation docs.

References
==========

1. `Shadow users
   <https://github.com/openstack/keystone-specs/blob/master/specs/keystone/newton/shadow-users-newton.rst>`_

2. `Shadow mapping
   <https://github.com/openstack/keystone-specs/blob/master/specs/keystone/ocata/shadow-mapping.rst>`_

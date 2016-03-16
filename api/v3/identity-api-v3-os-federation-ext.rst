OpenStack Identity API v3 OS-FEDERATION
=======================================

Provide the ability for users to manage Identity Providers (IdPs) and establish
a set of rules to map federation protocol attributes to Identity API
attributes. Requires v3.0+ of the Identity API.

What's New in Version 1.1
-------------------------

These features are considered stable as of September 4th, 2014.

- Introduced a mechanism to exchange an Identity Token for a SAML assertion.

- Introduced a mechanism to retrieve Identity Provider Metadata.

Definitions
-----------

- *Trusted Identity Provider*: An identity provider set up within the Identity
  API that is trusted to provide authenticated user information.

- *Service Provider*: A system entity that provides services to principals or
  other system entities, in this case, the OpenStack Identity API is the
  Service Provider.

- *Attribute Mapping*: The user information passed by a federation protocol for
  an already authenticated identity are called ``attributes``. Those
  ``attributes`` may not align 1:1 with the Identity API concepts. To help
  overcome such mismatches, a mapping can be done either on the sending side
  (third party identity provider), on the consuming side (Identity API
  service), or both.

What's New in Version 1.1
-------------------------

Corresponding to Identity API v3.3 release. These features are considered
stable as of September 4th, 2014.

- Deprecate list projects and domains in favour of core functionality available
  in Identity API v3.3.

What's New in Version 1.3
-------------------------

Corresponding to Identity API v3.5 release.

- Add Identity Provider specific websso routes.


API Resources
-------------

Identity Providers
~~~~~~~~~~~~~~~~~~

::

    /OS-FEDERATION/identity_providers

An Identity Provider is a third party service that is trusted by the Identity
API to authenticate identities.

Optional attributes:

- ``description`` (string)

  Describes the identity provider.

  If a value is not specified by the client, the service will default this
  value to ``null``.

- ``enabled`` (boolean)

  Indicates whether this identity provider should accept federated
  authentication requests.

  If a value is not specified by the client, the service will default this to
  ``false``.

- ``remote_ids`` (list)

  Valid remote IdP entity values from Identity Providers. If a value is not
  specified by the client, the list will be empty.


Protocols
~~~~~~~~~

::

    /OS-FEDERATION/identity_providers/{idp_id}/protocols

A protocol entry contains information that dictates which mapping rules to use
for a given incoming request. An IdP may have multiple supported protocols.

Required attributes:

- ``mapping_id`` (string)

  Indicates which mapping should be used to process federated authentication
  requests.

Mappings
~~~~~~~~

::

    /OS-FEDERATION/mappings

A ``mapping`` is a set of rules to map federation protocol attributes to
Identity API objects. An Identity Provider can have a single ``mapping``
specified per protocol. A mapping is simply a list of ``rules``. The only
Identity API objects that will support mapping are: ``group``.

Required attributes:

- ``rules`` (list of objects)

  Each object contains a rule for mapping attributes to Identity API concepts.
  A rule contains a ``remote`` attribute description and the destination
  ``local`` attribute.

- ``local`` (list of objects)

   References a local Identity API resource, such as a ``group`` or ``user`` to
   which the remote attributes will be mapped.

   Each object has one of two structures, as follows.

   To map a remote attribute value directly to a local attribute, identify the
   local resource type and attribute:

   ::

       {
           "user": {
               "name": "{0}"
           }
       }

   If the ``user`` attribute is missing when processing an assertion, server
   tries to directly map ``REMOTE_USER`` environment variable. If this variable
   is also unavailable the server returns an HTTP ``401 Unauthorized`` error.

   If the ``user`` has domain specified, the user is treated as existing in the
   backend, hence the server will fetch user details (id, name, roles, groups).

   If, however, the user does not exist in the backend, the server will
   respond with an appropriate HTTP error code.

   If no domain is specified in the local rule, user is deemed ephemeral
   and becomes a member of service domain named ``Federated``.

   An example of user object mapping to an existing user:

   ::

       {
            "user": {
                "name": "username"
                "domain": {
                    "name": "domain_name"
                }
            }
       }


   For attribute type and value mapping, identify the local resource type,
   attribute, and value:

   ::

       {
           "group": {
               "id": "89678b"
           }
       }

   This assigns authorization attributes, by way of role assignments on the
   specified group, to ephemeral users.

   ::

       {
           "group_ids": "{0}"
       }

   It is also possible to map multiple groups by providing a list of group ids.
   Those group ids can also be white/blacklisted.

- ``remote`` (list of objects)

  At least one object must be included.

  If more than one object is included, the local attribute is applied only if
  all remote attributes match.

  The value identified by ``type`` is always passed through unless a constraint
  is specified using either ``any_one_of`` or ``not_one_of``.

  - ``type`` (string)

    This represents an assertion type keyword.

  - ``any_one_of`` (list of strings)

    This is mutually exclusive with ``not_any_of``.

    The rule is matched only if any of the specified strings appear in the
    remote attribute ``type``.

  - ``not_any_of`` (list of strings)

    This is mutually exclusive with ``any_one_of``.

    The rule is not matched if any of the specified strings appear in the
    remote attribute ``type``.

  - ``regex`` (boolean)

    If ``true``, then each string will be evaluated as a `regular expression
    <http://docs.python.org/2/library/re.html>`__ search against the remote
    attribute ``type``.

Service Providers
~~~~~~~~~~~~~~~~~

::

    /OS-FEDERATION/service_providers

A service provider is a third party service that is trusted by the Identity
Service.

Required attributes:

- ``auth_url`` (string)

Specifies the protected URL where unscoped tokens can be retrieved once the
user is authenticated.

- ``sp_url`` (string)

Specifies the URL at the remote peer where assertion should be sent.

Optional attributes:

- ``description`` (string)

Describes the service provider

If a value is not specified by the client, the service may default this value
to ``null``.

- ``enabled`` (boolean)

Indicates whether bursting into this service provider is enabled by cloud
administrators. If set to ``false`` the SP will not appear in the catalog and
requests to generate an assertion will result in a 403 error.
If a value is not specified by the client, the service will default this to
``false``.

- ``relay_state_prefix`` (string)

Indicates the relay state prefix, used in the ECP wrapped SAML messages, by the
Service Provider.

If a value is not specified by the client, the service will default this value
to ``ss:mem:``.

Identity Provider API
---------------------

Register an Identity Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    PUT /OS-FEDERATION/identity_providers/{idp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider``

Request:

::

    {
        "identity_provider": {
            "description": "Stores ACME identities.",
            "remote_ids": ["acme_id_1", "acme_id_2"],
            "enabled": true
        }
    }

Response:

::

    Status: 201 Created

    {
        "identity_provider": {
            "description": "Stores ACME identities",
            "remote_ids": ["acme_id_1", "acme_id_2"],
            "enabled": true,
            "id": "ACME",
            "links": {
                "protocols": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME"
            }
        }
    }

List identity providers
~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/identity_providers

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_providers``

Response:

::

    Status: 200 OK

    {
        "identity_providers": [
            {
                "description": "Stores ACME identities",
                "remote_ids": ["acme_id_1", "acme_id_2"],
                "enabled": true,
                "id": "ACME",
                "links": {
                    "protocols": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols",
                    "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME"
                }
            },
            {
                "description": "Stores contractor identities",
                "remote_ids": ["sore_id_1", "store_id_2"],
                "enabled": false,
                "id": "ACME-contractors",
                "links": {
                    "protocols": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME-contractors/protocols",
                    "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME-contractors"
                }
            }
        ],
        "links": {
            "next": null,
            "previous": null,
            "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers"
        }
    }

Get Identity provider
~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/identity_providers/{idp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider``

Response:

::

    Status: 200 OK

    {
        "identity_provider": {
            "description": "Stores ACME identities",
            "remote_ids": ["acme_id_1", "acme_id_2"],
            "enabled": false,
            "id": "ACME",
            "links": {
                "protocols": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME"
            }
        }
    }

Delete identity provider
~~~~~~~~~~~~~~~~~~~~~~~~

::

    DELETE /OS-FEDERATION/identity_providers/{idp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider``

When an identity provider is deleted, any tokens generated by that identity
provider will be revoked.

Response:

::

    Status: 204 No Content

Update identity provider
~~~~~~~~~~~~~~~~~~~~~~~~

::

    PATCH /OS-FEDERATION/identity_providers/{idp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider``

Request:

::

    {
        "identity_provider": {
            "remote_ids": ["beta_id_1", "beta_id_2"],
            "enabled": true
        }
    }

Response:

::

    Status: 200 OK

    {
        "identity_provider": {
            "description": "Beta dev idp",
            "remote_ids": ["beta_id_1", "beta_id_2"],
            "enabled": true,
            "id": "ACME",
            "links": {
                "protocols": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME"
            }
        }
    }

When an identity provider is disabled, any tokens generated by that identity
provider will be revoked.

Add a protocol and attribute mapping to an identity provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    PUT /OS-FEDERATION/identity_providers/{idp_id}/protocols/{protocol_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocol``

Request:

::

    {
        "protocol": {
            "mapping_id": "xyz234"
        }
    }

Response:

::

    Status: 201 Created

     {
        "protocol": {
            "id": "saml2",
            "links": {
                "identity_provider": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols/saml2"
            },
            "mapping_id": "xyz234"
        }
    }

List all protocol and attribute mappings of an identity provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/identity_providers/{idp_id}/protocols

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocols``

Response:

::

    Status: 200 OK

    {
        "links": {
            "next": null,
            "previous": null,
            "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols"
        },
        "protocols": [
            {
                "id": "saml2",
                "links": {
                    "identity_provider": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME",
                    "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols/saml2"
                },
                "mapping_id": "xyz234"
            }
        ]
    }

Get a protocol and attribute mapping for an identity provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/identity_providers/{idp_id}/protocols/{protocol_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocol``

Response:

::

    Status: 200 OK

     {
        "protocol": {
            "id": "saml2",
            "links": {
                "identity_provider": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols/saml2"
            },
            "mapping_id": "xyz234"
        }
    }

Update the attribute mapping for an identity provider and protocol
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    PATCH /OS-FEDERATION/identity_providers/{idp_id}/protocols/{protocol_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocol``

Request:

::

    {
        "protocol": {
            "mapping_id": "xyz234"
        }
    }

Response:

::

    Status: 200 OK

     {
        "protocol": {
            "id": "saml2",
            "links": {
                "identity_provider": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME",
                "self": "http://identity:35357/v3/OS-FEDERATION/identity_providers/ACME/protocols/saml2"
            },
            "mapping_id": "xyz234"
        }
    }

Delete a protocol and attribute mapping from an identity provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    DELETE /OS-FEDERATION/identity_providers/{idp_id}/protocols/{protocol_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocol``

Response:

::

    Status: 204 No Content

Mapping API
-----------

Create a mapping
~~~~~~~~~~~~~~~~

::

    PUT /OS-FEDERATION/mappings/{mapping_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/mapping``

Request:

::

    {
        "mapping": {
            "rules": [
                {
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "group": {
                                "id": "0cd5e9"
                            }
                        }
                    ],
                    "remote": [
                        {
                            "type": "UserName"
                        },
                        {
                            "type": "orgPersonType",
                            "not_any_of": [
                                "Contractor",
                                "Guest"
                            ]
                        }
                    ]
                }
            ]
        }
    }

Response:

::

    Status: 201 Created

    {
        "mapping": {
            "id": "ACME",
            "links": {
                "self": "http://identity:35357/v3/OS-FEDERATION/mappings/ACME"
            },
            "rules": [
                {
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "group": {
                                "id": "0cd5e9"
                            }
                        }
                    ],
                    "remote": [
                        {
                            "type": "UserName"
                        },
                        {
                            "type": "orgPersonType",
                            "not_any_of": [
                                "Contractor",
                                "Guest"
                            ]
                        }
                    ]
                }
            ]
        }
    }

Get a mapping
~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/mappings/{mapping_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/mapping``

Response:

::

    Status: 200 OK

    {
        "mapping": {
            "id": "ACME",
            "links": {
                "self": "http://identity:35357/v3/OS-FEDERATION/mappings/ACME"
            },
            "rules": [
                {
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "group": {
                                "id": "0cd5e9"
                            }
                        }
                    ],
                    "remote": [
                        {
                            "type": "UserName"
                        },
                        {
                            "type": "orgPersonType",
                            "not_any_of": [
                                "Contractor",
                                "Guest"
                            ]
                        }
                    ]
                }
            ]
        }
    }

Update a mapping
~~~~~~~~~~~~~~~~

::

    PATCH /OS-FEDERATION/mappings/{mapping_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/mapping``

Request:

::

    {
        "mapping": {
            "rules": [
                {
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "group": {
                                "id": "0cd5e9"
                            }
                        }
                    ],
                    "remote": [
                        {
                            "type": "UserName"
                        },
                        {
                            "type": "orgPersonType",
                            "any_one_of": [
                                "Contractor",
                                "SubContractor"
                            ]
                        }
                    ]
                }
            ]
        }
    }

Response:

::

    Status: 200 OK

    {
        "mapping": {
            "id": "ACME",
            "links": {
                "self": "http://identity:35357/v3/OS-FEDERATION/mappings/ACME"
            },
            "rules": [
                {
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "group": {
                                "id": "0cd5e9"
                            }
                        }
                    ],
                    "remote": [
                        {
                            "type": "UserName"
                        },
                        {
                            "type": "orgPersonType",
                            "any_one_of": [
                                "Contractor",
                                "SubContractor"
                            ]
                        }
                    ]
                }
            ]
        }
    }

List all mappings
~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/mappings

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/mappings``

Response:

::

    Status 200 OK

    {
        "links": {
            "next": null,
            "previous": null,
            "self": "http://identity:35357/v3/OS-FEDERATION/mappings"
        },
        "mappings": [
            {
                "id": "ACME",
                "links": {
                    "self": "http://identity:35357/v3/OS-FEDERATION/mappings/ACME"
                },
                "rules": [
                    {
                        "local": [
                            {
                                "user": {
                                    "name": "{0}"
                                }
                            },
                            {
                                "group": {
                                    "id": "0cd5e9"
                                }
                            }
                        ],
                        "remote": [
                            {
                                "type": "UserName"
                            },
                            {
                                "type": "orgPersonType",
                                "any_one_of": [
                                    "Contractor",
                                    "SubContractor"
                                ]
                            }
                        ]
                    }
                ]
            }
        ]
    }

Delete a mapping
~~~~~~~~~~~~~~~~

::

    DELETE /OS-FEDERATION/mappings/{mapping_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/mapping``

Response:

::

    Status: 204 No Content

Service Provider API
--------------------

Register a Service Provider
~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    PUT /OS-FEDERATION/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/service_provider``


Request:

::

    {
        "service_provider": {
            "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
            "description": "Remote Service Provider",
            "enabled": true,
            "sp_url": "https://example.com:5000/Shibboleth.sso/SAML2/ECP"
        }
    }

Response:

::

    Status 201 Created

    {
        "service_provider": {
            "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
            "description": "Remote Service Provider",
            "enabled": true,
            "id": "ACME",
            "links": {
                "self": "https://identity:35357/v3/OS-FEDERATION/service_providers/ACME"
            },
            "relay_state_prefix": "ss:mem:",
            "sp_url": "https://example.com:5000/Shibboleth.sso/SAML2/ECP"
        }
    }

Listing Service Providers
~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/service_providers

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/service_providers``


Response:

::

    Status: 200 OK

    {
        "links": {
            "next": null,
            "previous": null,
            "self": "http://identity:35357/v3/OS-FEDERATION/service_providers"
        },
        "service_providers": [
            {
                "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
                "description": "Stores ACME identities",
                "enabled": true,
                "id": "ACME",
                "links": {
                    "self": "http://identity:35357/v3/OS-FEDERATION/service_providers/ACME"
                },
                "relay_state_prefix": "ss:mem:",
                "sp_url": "https://example.com:5000/Shibboleth.sso/SAML2/ECP"
            },
            {
                "auth_url": "https://other.example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
                "description": "Stores contractor identities",
                "enabled": false,
                "id": "ACME-contractors",
                "links": {
                    "self": "http://identity:35357/v3/OS-FEDERATION/service_providers/ACME-contractors"
                },
                "relay_state_prefix": "ss:mem:",
                "sp_url": "https://other.example.com:5000/Shibboleth.sso/SAML2/ECP"
            }
        ]
    }

Get Service Provider
~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/service_provider``

Response:

::

    Status 200 OK

    {
        "service_provider": {
            "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
            "description": "Remote Service Provider",
            "enabled": true,
            "id": "ACME",
            "links": {
                "self": "https://identity:35357/v3/OS-FEDERATION/service_providers/ACME"
            },
            "relay_state_prefix": "ss:mem:",
            "sp_url": "https://example.com:5000/Shibboleth.sso/SAML2/ECP"
        }
    }

Delete Service Provider
~~~~~~~~~~~~~~~~~~~~~~~~

::

    DELETE /OS-FEDERATION/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/service_provider``


Response:

::

    Status: 204 No Content

Update Service Provider
~~~~~~~~~~~~~~~~~~~~~~~~

::

    PATCH /OS-FEDERATION/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/service_provider``

Request:

::

    {
        "service_provider": {
            "auth_url": "https://new.example.com:5000/v3/OS-FEDERATION/identity_providers/protocol/saml2/auth",
            "enabled": true,
            "relay_state_prefix": "ss:temp:",
            "sp_auth": "https://new.example.com:5000/Shibboleth.sso/SAML2/ECP"
        }
    }

Response:

::

    Status 200 OK

    {
        "service_provider": {
            "auth_url": "https://new.example.com:5000/v3/OS-FEDERATION/identity_providers/protocol/saml2/auth",
            "description": "Remote Service Provider",
            "enabled": true,
            "id": "ACME",
            "links": {
                "self": "https://identity:35357/v3/OS-FEDERATION/service_providers/ACME"
            },
            "relay_state_prefix": "ss:temp:",
            "sp_url": "https://new.example.com:5000/Shibboleth.sso/SAML2/ECP"
        }
    }


Listing projects and domains
----------------------------

**Deprecated in v1.1**. This section is deprecated as the functionality is
available in the core Identity API.

List projects a federated user can access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/projects

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/projects``

**Deprecated in v1.1**. Use core ``GET /auth/projects``. This call has the same
response format.

Returns a collection of projects to which the federated user has authorization
to access. To access this resource, an unscoped token is used, the user can
then select a project and request a scoped token. Note that only enabled
projects will be returned.

Response:

::

    Status: 200 OK

    {
        "projects": [
            {
                "domain_id": "37ef61",
                "enabled": true,
                "id": "12d706",
                "links": {
                    "self": "http://identity:35357/v3/projects/12d706"
                },
                "name": "a project name"
            },
            {
                "domain_id": "37ef61",
                "enabled": true,
                "id": "9ca0eb",
                "links": {
                    "self": "http://identity:35357/v3/projects/9ca0eb"
                },
                "name": "another project"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-FEDERATION/projects",
            "previous": null,
            "next": null
        }
    }

List domains a federated user can access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/domains

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/domains``

**Deprecated in v1.1**. Use core ``GET /auth/domains``. This call has the same
response format.

Returns a collection of domains to which the federated user has authorization
to access. To access this resource, an unscoped token is used, the user can
then select a domain and request a scoped token. Note that only enabled domains
will be returned.

Response:

::

    Status: 200 OK

    {
        "domains": [
            {
                "description": "desc of domain",
                "enabled": true,
                "id": "37ef61",
                "links": {
                    "self": "http://identity:35357/v3/domains/37ef61"
                },
                "name": "my domain"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-FEDERATION/domains",
            "previous": null,
            "next": null
        }
    }

Example Mapping Rules
---------------------

Map identities to their own groups
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is an example of *Attribute type and value mappings*, where an attribute
type and value are mapped into an Identity API property and value.

::

    {
        "rules": [
            {
                "local": [
                    {
                        "user": {
                            "name": "{0}"
                        }
                    }
                ],
                "remote": [
                    {
                        "type": "UserName"
                    }
                ]
            },
            {
                "local": [
                    {
                        "group": {
                            "id": "0cd5e9"
                        }
                    }
                ],
                "remote": [
                    {
                        "type": "orgPersonType",
                        "not_any_of": [
                            "Contractor",
                            "SubContractor"
                        ]
                    }
                ]
            },
            {
                "local": [
                    {
                        "group": {
                            "id": "85a868"
                        }
                    }
                ],
                "remote": [
                    {
                        "type": "orgPersonType",
                        "any_one_of": [
                            "Contractor",
                            "SubContractor"
                        ]
                    }
                ]
            }
        ]
    }

Find specific users, set them to admin group
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is an example that is similar to the previous, but displays how multiple
``remote`` properties can be used to narrow down on a property.

::

    {
        "rules": [
            {
                "local": [
                    {
                        "user": {
                            "name": "{0}"
                        }
                    },
                    {
                        "group": {
                            "id": "85a868"
                        }
                    }
                ],
                "remote": [
                    {
                        "type": "UserName"
                    },
                    {
                        "type": "orgPersonType",
                        "any_one_of": [
                            "Employee"
                        ]
                    },
                    {
                        "type": "sn",
                        "any_one_of": [
                            "Young"
                        ]
                    }
                ]
            }
        ]
    }

Map identities to multiple groups without domain reference
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This example shows how to map a user to multiple groups (without domain
reference) using the ``group_ids`` attribute. Those group ids can also be
white/blacklisted.

::

    {
        "rules": [
            {
                "local": [
                    {
                        "user": {
                            "name": "{0}"
                        }
                    },
                    {
                        "group_ids": "{1}"
                    }
                ],
                "remote": [
                    {
                        "type": "UserName"
                    },
                    {
                        "type": "group_ids",
                        "whitelist": [
                            "abc123;def456"
                        ]
                    }
                ]
            }
        ]
    }


Authenticating
--------------

Request an unscoped OS-FEDERATION token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET/POST /OS-FEDERATION/identity_providers/{identity_provider}/protocols/{protocol}/auth

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/identity_provider_protocol_auth``

A federated ephemeral user may request an unscoped token, which can be used to
get a scoped token.

If the user is mapped directly (mapped to an existing user), a standard,
unscoped token will be issued.

Due to the fact that this part of authentication is strictly connected with the
SAML2 authentication workflow, a client should not send any data, as the
content may be lost when a client is being redirected between Service Provider
and Identity Provider. Both HTTP methods - GET and POST should be allowed as
Web Single Sign-On (WebSSO) and Enhanced Client Proxy (ECP) mechanisms have
different authentication workflows and use different HTTP methods while
accessing protected endpoints.

The returned token will contain information about the groups to which the
federated user belongs.

Example Identity API token response: `Various OpenStack token responses
<identity-api-v3.md#authentication-responses>`__

Example of an OS-FEDERATION token:

::

    {
        "token": {
            "methods": [
                "mapped"
            ],
            "user": {
                "domain": {
                    "id": "Federated"
                },
                "id": "username%40example.com",
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

Request a scoped OS-FEDERATION token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    POST /auth/tokens

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/rel/auth_tokens``

A federated user may request a scoped token, by using the unscoped token. A
project or domain may be specified by either id or name. An id is sufficient to
uniquely identify a project or domain.

Example request:

::

    {
        "auth": {
            "identity": {
                "methods": [
                    "token"
                ],
                "token: {
                    "id": "--federated-token-id--"
                }
            }
        },
        "scope": {
            "project": {
                "id": "263fd9"
            }
        }
    }

Similarly to the returned unscoped token, the returned scoped token will have
an ``OS-FEDERATION`` section added to the ``user`` portion of the token.

Example of an OS-FEDERATION token:

::

    {
        "token": {
            "methods": [
                "token"
            ],
            "roles": [
                {
                    "id": "36a8989f52b24872a7f0c59828ab2a26",
                    "name": "admin"
                }
            ],
            "expires_at": "2014-08-06T13:43:43.367202Z",
            "project": {
                "domain": {
                    "id": "1789d1",
                    "links": {
                        "self": "http://identity:35357/v3/domains/1789d1"
                    },
                    "name": "example.com"
                },
                "id": "263fd9",
                "links": {
                    "self": "http://identity:35357/v3/projects/263fd9"
                },
                "name": "project-x"
            },
            "catalog": [
                {
                    "endpoints": [
                        {
                            "id": "39dc322ce86c4111b4f06c2eeae0841b",
                            "interface": "public",
                            "region": "RegionOne",
                            "url": "http://localhost:5000"
                        },
                        {
                            "id": "ec642f27474842e78bf059f6c48f4e99",
                            "interface": "internal",
                            "region": "RegionOne",
                            "url": "http://localhost:5000"
                        },
                        {
                            "id": "c609fc430175452290b62a4242e8a7e8",
                            "interface": "admin",
                            "region": "RegionOne",
                            "url": "http://localhost:35357"
                        }
                    ],
                    "id": "266c2aa381ea46df81bb05ddb02bd14a",
                    "name": "keystone",
                    "type": "identity"
                }
            ],
            "user": {
                "domain": {
                    "id": "Federated"
                },
                "id": "username%40example.com",
                "name": "username@example.com",
                "OS-FEDERATION": {
                    "identity_provider": "ACME",
                    "protocol": "SAML",
                    "groups": [
                        {"id": "abc123"},
                        {"id": "bcd234"}
                    ]
                }
            },
            "issued_at": "2014-08-06T12:43:43.367288Z"
        }
    }

Web Single Sign On authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*New in version 1.2*

::

    GET /auth/OS-FEDERATION/websso/{protocol}?origin=https%3A//horizon.example.com

For Web Single Sign On authentication, users are expected to enter another
URL endpoint. Upon successful authentication, instead of issuing a standard
unscoped token, Keystone will issue JavaScript code that redirects the web
browser to the originating Horizon. An unscoped federated token will be
included in the form being sent.


*New in version 1.3*

::

    GET /auth/OS-FEDERATION/identity_providers/{idp_id}/protocol/{protocol_id}/websso?origin=https%3A//horizon.example.com


In contrast to the above route, this route begins a Web Single Sign On request
that is specific to the supplied Identity Provider and Protocol. Keystone will
issue JavaScript that handles redirections in the same way as the other route.
An unscoped federated token will be included in the form being sent.

Generating Assertions
---------------------

*New in version 1.1*

Generate a SAML assertion
~~~~~~~~~~~~~~~~~~~~~~~~~

::

    POST /auth/OS-FEDERATION/saml2

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/saml2``

A user may generate a SAML assertion document based on the scoped token that is
used in the request.

Request Parameters:

To generate a SAML assertion, a user must provides a scoped token ID and
Service Provider ID in the request body.

Example request:

::

    {
        "auth": {
            "identity": {
                "methods": [
                    "token"
                ],
                "token": {
                    "id": "--token_id--"
                }
            },
            "scope": {
                "service_provider": {
                    "id": "--sp_id--"
                }
            }
        }
    }

The response will be a full SAML assertion. Note that for readability the
certificate has been truncated. Server will also set two HTTP headers:
``X-sp-url`` and ``X-auth-url``. The former is the URL where assertion should
be sent, whereas the latter remote URL where token will be issued once the
client is finally authenticated.

Response:

::

    Headers:
        Content-Type: text/xml
        X-sp-url: http://beta.example.com/Shibboleth.sso/POST/ECP
        X-auth-url: http://beta.example.com:5000/v3/OS-FEDERATION/identity_providers/beta/protocols/auth

    <?xml version="1.0" encoding="UTF-8"?>
    <ns0:Response xmlns:ns0="urn:oasis:names:tc:SAML:2.0:protocol" xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion" xmlns:xmldsig="http://www.w3.org/2000/09/xmldsig#" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" Destination="http://beta.example.com/Shibboleth.sso/POST/ECP" ID="818dee98a5d44a238ae3038d26cbebb6" IssueInstant="2015-05-27T13:23:48Z" Version="2.0">
    <saml:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity">http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:Issuer>
    <ns0:Status>
        <ns0:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
    </ns0:Status>
    <saml:Assertion ID="68237000470e47a690bdd513bb264460" IssueInstant="2015-05-27T13:23:47Z" Version="2.0">
        <saml:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity">http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:Issuer>
        <xmldsig:Signature>
            <xmldsig:SignedInfo>
                <xmldsig:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                <xmldsig:SignatureMethod Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/>
                <xmldsig:Reference URI="#68237000470e47a690bdd513bb264460">
                    <xmldsig:Transforms>
                        <xmldsig:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
                        <xmldsig:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                    </xmldsig:Transforms>
                    <xmldsig:DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/>
                    <xmldsig:DigestValue>IgfoWcCoBpmv64ianaK/qj63QQQ=</xmldsig:DigestValue>
                </xmldsig:Reference>
            </xmldsig:SignedInfo>
            <xmldsig:SignatureValue>H6GvkAcDW0BSoBaktpVTxUFtvUAcFMXRqYXLFvmse5DeOSnByvGOgW/yJMjIqzwG
            LjCqJXYMePIkEUYb4kqbbkN1wNFuxKtmACcC3T3/7rAavrIz3I4cT6mCipN9qFlE
            tzR0mD2IZhExuTzyMaON8krTWWoddx8LIYEfQ03O4eSYObi5fHmGJRGs9D5De0aK
            XkIeKo7HRAjZsU5fAMGlEKfazemTZMBbnpUD//oFsxf1yFcFTOyiAHddAaG7Rqv3
            4SYjYo4dRKAI/yQuA+MVmHDcJUE+KVqVoJZJSVJe+Lz+X1ReRlEgvP0mhaM0yY+R
            w7FozqQyKSKJW9abmxJTFQ==</xmldsig:SignatureValue>
            <xmldsig:KeyInfo>
                <xmldsig:X509Data>
                    <xmldsig:X509Certificate>...</xmldsig:X509Certificate>
                </xmldsig:X509Data>
            </xmldsig:KeyInfo>
        </xmldsig:Signature>
        <saml:Subject>
            <saml:NameID>admin</saml:NameID>
            <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                <saml:SubjectConfirmationData NotOnOrAfter="2015-05-27T14:23:47.711682Z" Recipient="http://beta.example.com/Shibboleth.sso/POST/ECP/">
            </saml:SubjectConfirmation>
        </saml:Subject>
        <saml:AuthnStatement AuthnInstant="2015-05-27T13:23:47Z" SessionIndex="cd839a3ff0fc4a4aab52e55fae8094a2" SessionNotOnOrAfter="2015-05-27T14:23:47.711682Z">
            <saml:AuthnContext>
                <saml:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:Password</saml:AuthnContextClassRef>
                <saml:AuthenticatingAuthority>http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:AuthenticatingAuthority>
            </saml:AuthnContext>
        </saml:AuthnStatement>
        <saml:AttributeStatement>
            <saml:Attribute Name="openstack_user" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
            </saml:Attribute>
            <saml:Attribute Name="openstack_user_domain" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                <saml:AttributeValue xsi:type="xs:string">Default</saml:AttributeValue>
            </saml:Attribute>
            <saml:Attribute Name="openstack_roles" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
            </saml:Attribute>
            <saml:Attribute Name="openstack_project" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
            </saml:Attribute>
            <saml:Attribute Name="openstack_project_domain" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                <saml:AttributeValue xsi:type="xs:string">Default</saml:AttributeValue>
            </saml:Attribute>
        </saml:AttributeStatement>
    </saml:Assertion>
    </ns0:Response>

For more information about how a SAML assertion is structured, refer to the
`specification <http://saml.xml.org/saml-specifications>`__.

Generate an ECP wrapped SAML assertion
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    POST /auth/OS-FEDERATION/saml2/ecp

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/saml2/ecp``

A user may generate a SAML assertion document to work with the
*Enhanced Client or Proxy* (ECP) profile based on the scoped token that is
used in the request.

Request Parameters:

To generate an ECP wrapped SAML assertion, a user must provides a scoped token
ID and Service Provider ID in the request body.

Example request:

::

    {
        "auth": {
            "identity": {
                "methods": [
                    "token"
                ],
                "token": {
                    "id": "--token_id--"
                }
            },
            "scope": {
                "service_provider": {
                    "id": "--sp_id--"
                }
            }
        }
    }

The response will be an ECP wrapped SAML assertion. Note that for readability
the certificate has been truncated. Server will also set two HTTP headers:
``X-sp-url`` and ``X-auth-url``. The former is the URL where assertion should
be sent, whereas the latter remote URL where token will be issued once the
client is finally authenticated.

::

    Headers:
        Content-Type: text/xml
        X-sp-url: http://beta.example.com/Shibboleth.sso/POST/ECP
        X-auth-url: http://beta.example.com:5000/v3/OS-FEDERATION/identity_providers/beta/protocols/auth

    <?xml version='1.0' encoding='UTF-8'?>
    <ns0:Envelope
        xmlns:ns0="http://schemas.xmlsoap.org/soap/envelope/"
        xmlns:ns1="urn:oasis:names:tc:SAML:2.0:profiles:SSO:ecp"
        xmlns:ns2="urn:oasis:names:tc:SAML:2.0:protocol"
        xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
        xmlns:xmldsig="http://www.w3.org/2000/09/xmldsig#"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <ns0:Header>
            <ns1:RelayState ns0:actor="http://schemas.xmlsoap.org/soap/actor/next" ns0:mustUnderstand="1">ss:mem:1ddfe8b0f58341a5a840d2e8717b0737</ns1:RelayState>
        </ns0:Header>
        <ns0:Body>
            <ns2:Response Destination="http://beta.example.com/Shibboleth.sso/POST/ECP" ID="8c21de08d2f2435c9acf13e72c982846" IssueInstant="2015-03-25T14:43:21Z" Version="2.0">
                <saml:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity">http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:Issuer>
                <ns2:Status>
                    <ns2:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
                </ns2:Status>
                <saml:Assertion ID="a5f02efb0bff4044b294b4583c7dfc5d" IssueInstant="2015-03-25T14:43:21Z" Version="2.0">
                    <saml:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity">http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:Issuer>
                    <xmldsig:Signature>
                        <xmldsig:SignedInfo>
                            <xmldsig:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
                            <xmldsig:SignatureMethod Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1" />
                            <xmldsig:Reference URI="#a5f02efb0bff4044b294b4583c7dfc5d">
                                <xmldsig:Transforms>
                                    <xmldsig:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
                                    <xmldsig:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
                                </xmldsig:Transforms>
                                <xmldsig:DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha1" />
                                <xmldsig:DigestValue>0KH2CxdkfzU+6eiRhTC+mbObUKI=</xmldsig:DigestValue>
                            </xmldsig:Reference>
                        </xmldsig:SignedInfo>
                        <xmldsig:SignatureValue>m2jh5gDvX/1k+4uKtbb08CHp2b9UWsLwjtMijs9C9gZV2dIJKiF9SJBWE4C79qT4
    uktgeB0RQiFrgxOGfpp1gyQunmNyZcipcetOk4PebH4/z+po/59w8oGp89fPfdRj
    WhWA0fWP32Pr5eslRQjbHnSRTFMp3ycBZHsCCsTWdhyiWC6aERsspHeeGjkzxRAZ
    HxJ8oLMj/TWBJ2iaUDUT6cxa1svmtumoC3GPPOreuGELXTL5MtKotTVqYN6lZP8B
    Ueaji11oRI1HE9XMuPu0iYlSo1i3JyejciSFgplgdHsebpM29PMo8oz2TCybY39p
    kmuD4y9XX3lRBcpJRxku7w==</xmldsig:SignatureValue>
                        <xmldsig:KeyInfo>
                            <xmldsig:X509Data>
                                <xmldsig:X509Certificate>...</xmldsig:X509Certificate>
                            </xmldsig:X509Data>
                        </xmldsig:KeyInfo>
                    </xmldsig:Signature>
                    <saml:Subject>
                        <saml:NameID>admin</saml:NameID>
                        <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                            <saml:SubjectConfirmationData NotOnOrAfter="2015-03-25T15:43:21.172385Z" Recipient="http://beta.example.com/Shibboleth.sso/POST/ECP" />
                        </saml:SubjectConfirmation>
                    </saml:Subject>
                    <saml:AuthnStatement AuthnInstant="2015-03-25T14:43:21Z" SessionIndex="9790eb729858456f8a33b7a11f0a637e" SessionNotOnOrAfter="2015-03-25T15:43:21.172385Z">
                        <saml:AuthnContext>
                            <saml:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:Password</saml:AuthnContextClassRef>
                            <saml:AuthenticatingAuthority>http://keystone.idp/v3/OS-FEDERATION/saml2/idp</saml:AuthenticatingAuthority>
                        </saml:AuthnContext>
                    </saml:AuthnStatement>
                    <saml:AttributeStatement>
                        <saml:Attribute Name="openstack_user" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                            <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
                        </saml:Attribute>
                        <saml:Attribute Name="openstack_user_domain" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                            <saml:AttributeValue xsi:type="xs:string">Default</saml:AttributeValue>
                        </saml:Attribute>
                        <saml:Attribute Name="openstack_roles" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                            <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
                        </saml:Attribute>
                        <saml:Attribute Name="openstack_project" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                            <saml:AttributeValue xsi:type="xs:string">admin</saml:AttributeValue>
                        </saml:Attribute>
                        <saml:Attribute Name="openstack_project_domain" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
                            <saml:AttributeValue xsi:type="xs:string">Default</saml:AttributeValue>
                        </saml:Attribute>
                    </saml:AttributeStatement>
                </saml:Assertion>
            </ns2:Response>
        </ns0:Body>
    </ns0:Envelope>


Retrieve Metadata properties
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    GET /OS-FEDERATION/saml2/metadata

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-FEDERATION/1.0/rel/metadata``

A user may retrieve Metadata about an Identity Service acting as an Identity
Provider.

The response will be a full document with Metadata properties. Note that for
readability, this example certificate has been truncated.

Response:

::

    Headers:
        Content-Type: text/xml

    <?xml version="1.0" encoding="UTF-8"?>
    <ns0:EntityDescriptor xmlns:ns0="urn:oasis:names:tc:SAML:2.0:metadata"
       xmlns:ns1="http://www.w3.org/2000/09/xmldsig#" entityID="k2k.com/v3/OS-FEDERATION/idp"
       validUntil="2014-08-19T21:24:17.411289Z">
      <ns0:IDPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
        <ns0:KeyDescriptor use="signing">
          <ns1:KeyInfo>
            <ns1:X509Data>
              <ns1:X509Certificate>MIIDpTCCAo0CAREwDQYJKoZIhvcNAQEFBQAwgZ</ns1:X509Certificate>
            </ns1:X509Data>
          </ns1:KeyInfo>
        </ns0:KeyDescriptor>
      </ns0:IDPSSODescriptor>
      <ns0:Organization>
        <ns0:OrganizationName xml:lang="en">openstack</ns0:OrganizationName>
        <ns0:OrganizationDisplayName xml:lang="en">openstack</ns0:OrganizationDisplayName>
        <ns0:OrganizationURL xml:lang="en">openstack</ns0:OrganizationURL>
      </ns0:Organization>
      <ns0:ContactPerson contactType="technical">
        <ns0:Company>openstack</ns0:Company>
        <ns0:GivenName>first</ns0:GivenName>
        <ns0:SurName>lastname</ns0:SurName>
        <ns0:EmailAddress>admin@example.com</ns0:EmailAddress>
        <ns0:TelephoneNumber>555-555-5555</ns0:TelephoneNumber>
      </ns0:ContactPerson>
    </ns0:EntityDescriptor>

For more information about how a SAML assertion is structured, refer to the
`specification <http://saml.xml.org/saml-specifications>`__.

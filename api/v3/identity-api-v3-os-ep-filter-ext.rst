OpenStack Identity API v3 OS-EP-FILTER Extension
================================================

This extension enables creation of ad-hoc catalogs for each project-scoped
token request. To do so, this extension uses either static project-endpoint
associations or dynamic custom endpoints groups to associate service endpoints
with projects.

What's New in Version 1.1
-------------------------

These features are not yet considered stable (expected September 4th,
2014).

-  Introduced support for Endpoint Groups

API Resources
-------------

*New in version 1.1*

Endpoint Group
~~~~~~~~~~~~~~

Represents a dynamic collection of service endpoints having the same
characteristics, such as service\_id, interface, or region. Indeed, any
endpoint attribute could be used as part of a filter.

A classic use case is to filter endpoints based on region. For example, suppose
a user wants to filter service endpoints returned in the service catalog by
region, the following endpoint group may be used:

::

    {
        "endpoint_group": {
            "description": "Example Endpoint Group",
            "filters": {
                "region_id": "e68c72"
            },
            "name": "EP-GROUP-1"
        }
    }

This implies an Endpoint Group with filtering criteria of the form:

::

    region_id = "e68c72"

API
---

Project-Endpoint Associations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If a valid X-Auth-Token token in not present in the HTTP header and/or the user
does not have the right authorization an HTTP ``401 Unauthorized`` is returned.

Create Association
^^^^^^^^^^^^^^^^^^

::

    PUT /OS-EP-FILTER/projects/{project_id}/endpoints/{endpoint_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoint``

Modifies the endpoint resource adding an association between the project and
the endpoint.

Response:

::

    Status: 204 No Content

Check Association
^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/projects/{project_id}/endpoints/{endpoint_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoint``

Verifies the existence of an association between a project and an endpoint.

Response:

::

    Status: 204 No Content

List Associations for Project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/projects/{project_id}/endpoints

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoints``

Returns all the endpoints that are currently associated with a specific
project.

Response:

::

    Status: 200 OK
    {
        "endpoints": [
            {
                "id": "6fedc0",
                "interface": "public",
                "url": "http://identity:35357/",
                "region": "north",
                "links": {
                    "self": "http://identity:35357/v3/endpoints/6fedc0"
                },
                "service_id": "1b501a"
            },
            {
                "id": "6fedc0",
                "interface": "internal",
                "region": "south",
                "url": "http://identity:35357/",
                "links": {
                    "self": "http://identity:35357/v3/endpoints/6fedc0"
                },
                "service_id": "1b501a"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-EP-FILTER/projects/{project_id}/endpoints",
            "previous": null,
            "next": null
        }
    }

Delete Association
^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/projects/{project_id}/endpoints/{endpoint_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoint``

Eliminates a previously created association between a project and an endpoint.

Response:

::

    Status: 204 No Content

Get projects associated with endpoint
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoints/{endpoint_id}/projects

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_projects``

Returns a list of projects that are currently associated with the given
endpoint.

Response:

::

    Status: 200 OK

    {
        "projects": [
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "263fd9",
                "links": {
                     "self": "http://identity:35357/v3/projects/263fd9"
                },
                "name": "a project name 1",
                "description": "a project description 1"
            },
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "61a1b7",
                "links": {
                     "self": "http://identity:35357/v3/projects/61a1b7"
                },
                "name": "a project name 2",
                "description": "a project description 2"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-EP-FILTER/endpoints/6fedc0/projects",
            "previous": null,
            "next": null
        }

}

Endpoint Groups
~~~~~~~~~~~~~~~

*New in version 1.1*

Required attributes:

- ``name`` (string)

User-facing name of the service.

- ``filters`` (object)

  Describes the filtering performed by the endpoint group. The filter used must
  be an ``endpoint`` property, such as ``interface``, ``service_id``,
  ``region_id`` and ``enabled``. Note that if using ``interface`` as a filter,
  the only available values are ``public``, ``internal`` and ``admin``.

Optional attributes:

- ``description`` (string)

  User-facing description of the service.

Create Endpoint Group Filter
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    POST /OS-EP-FILTER/endpoint_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_groups``

Request:

::

    {
        "endpoint_group": {
            "description": "endpoint group description",
            "filters": {
                "interface": "admin",
                "service_id": "1b501a"
            }
            "name": "endpoint group name",
        }
    }

Response:

::

    Status: 201 Created

    {
        "endpoint_group": {
            "description": "endpoint group description",
             "filters": {
                "interface": "admin",
                "service_id": "1b501a"
            },
            "id": "ac4861",
            "links": {
                "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/ac4861"
            },
            "name": "endpoint group name"
        }
    }

Get Endpoint Group
^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group``

Response:

::

    Status: 200 OK

    {
        "endpoint_group": {
            "description": "endpoint group description",
            "filters": {
                "interface": "admin",
                "service_id": "1b501a"
            },
            "id": "ac4861",
            "links": {
                "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/ac4861"
            },
            "name": "endpoint group name"
        }
    }

Check Endpoint Group
^^^^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group``

Response:

::

    Status: 200 OK

Update Endpoint Group
^^^^^^^^^^^^^^^^^^^^^

::

    PATCH /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group``

The request block is the same as the one for create endpoint group, except that
only the attributes that are being updated need to be included.

Request:

::

    {
        "endpoint_group": {
            "description": "endpoint group description",
            "filters": {
                "interface": "admin",
                "service_id": "1b501a"
            },
            "name": "endpoint group name"
        }
    }

Response:

::

    Status: 200 OK

    {
        "endpoint_group": {
            "description": "endpoint group description",
            "filters": {
                "interface": "admin",
                "service_id": "1b501a"
            },
            "id": "ac4861",
            "links": {
                "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/ac4861"
            },
            "name": "endpoint group name"
        }
    }

Remove Endpoint Group
^^^^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group``

Response:

::

    Status: 204 No Content

List All Endpoint Groups
^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoint_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_groups``

Response:

::

    Status: 200 OK

    {
        "endpoint_groups": [
            {
                "endpoint_group": {
                    "description": "endpoint group description #1",
                    "filters": {
                        "interface": "admin",
                        "service_id": "1b501a"
                    },
                    "id": "ac4861",
                    "links": {
                        "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/ac4861"
                    },
                    "name": "endpoint group name #1"
                }
            },
            {
                "endpoint_group": {
                    "description": "endpoint group description #2",
                    "filters": {
                        "interface": "admin"
                    },
                    "id": "3de68c",
                    "links": {
                        "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/3de68c"
                    },
                    "name": "endpoint group name #2"
                }
            }
        ],
        "links": {
            "self": "https://identity:35357/v3/OS-EP-FILTER/endpoint_groups",
            "previous": null,
            "next": null
        }
    }

List Endpoint Groups Associated with Project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/projects/{project_id}/endpoint_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoint_groups``

Response:

::

    Status: 200 OK

    {
        "endpoint_groups": [
            {
                "endpoint_group": {
                    "description": "endpoint group description #1",
                    "filters": {
                        "interface": "admin",
                        "service_id": "1b501a"
                    },
                    "id": "ac4861",
                    "links": {
                        "self": "http://localhost:35357/v3/OS-EP-FILTER/endpoint_groups/ac4861"
                    },
                    "name": "endpoint group name #1"
                }
            }
        ],
        "links": {
            "self": "https://identity:35357/v3/OS-EP-FILTER/endpoint_groups",
            "previous": null,
            "next": null
        }
    }

Project to Endpoint Group Relationship
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create Endpoint Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    PUT /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_project``

Response:

::

    Status: 204 No Content

Get Endpoint Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_project``

Response:

::

    Status: 200 OK

    {
        "project": {
            "domain_id": "1789d1",
            "enabled": true,
            "id": "263fd9",
            "links": {
                "self": "http://identity:35357/v3/projects/263fd9"
            },
            "name": "project name #1",
            "description": "project description #1"
        }
    }

Check Endpoint Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_project``

Response:

::

    Status: 200 OK

Delete Endpoint Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_project``

Response:

::

    Status: 204 No Content

List Projects Associated with Endpoint Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_projects``

Response:

::

    Status: 200 OK

    {
        "projects": [
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "263fd9",
                "links": {
                     "self": "http://identity:35357/v3/projects/263fd9"
                },
                "name": "a project name 1",
                "description": "a project description 1"
            },
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "61a1b7",
                "links": {
                     "self": "http://identity:35357/v3/projects/61a1b7"
                },
                "name": "a project name 2",
                "description": "a project description 2"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/projects",
            "previous": null,
            "next": null
        }
    }

List Service Endpoints Associated with Endpoint Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/endpoints

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/endpoint_group_endpoints``

Response:

::

    Status: 200 OK

    {
        "endpoints": [
            {
                "enabled": true,
                "id": "6fedc0"
                "interface": "admin",
                "legacy_endpoint_id": "6fedc0",
                "links": {
                    "self": "http://identity:35357/v3/endpoints/6fedc0"
                },
                "region": "RegionOne",
                "service_id": "1b501a",
                "url": "http://localhost:9292"
            },
            {
                "enabled": true,
                "id": "b501aa"
                "interface": "internal",
                "legacy_endpoint_id": "b501aa",
                "links": {
                    "self": "http://identity:35357/v3/endpoints/b501aa"
                },
                "region": "RegionOne",
                "service_id": "1b501a",
                "url": "http://localhost:9292"
            },
            {
                "enabled": true,
                "id": "b7c573"
                "interface": "public",
                "legacy_endpoint_id": "b7c573",
                "links": {
                    "self": "http://identity:35357/v3/endpoints/b7c573"
                },
                "region": "RegionOne",
                "service_id": "1b501a",
                "url": "http://localhost:9292"
            }
        ],
        "links": {
            "self": "http://identity:35357/v3/OS-EP-FILTER/endpoint_groups/{endpoint_group_id}/endpoints",
            "previous": null,
            "next": null
        }
    }


Service Providers filtering
~~~~~~~~~~~~~~~~~~~~~~~~~~~

With Keystone2Keystone capabilities enabled each token response includes
top-level entity ``service_providers`` with list of all trusted service
providers configured with the OpenStack deployment. Service Providers filtering
enables associating service providers with projects. As a result users will
get only those associated service providers instead of full list of available
(and enabled) service providers. For the sake of operability, an administrator
can also create ``Service Provider group`` specyfing multiple service
providers, and associate the group with a project. This will require two API
calls (creating a group with n service providers in it and associating the
group with a project) as opposed to n API calls (each call for associating a
service provider with project).

Create Association
^^^^^^^^^^^^^^^^^^

::

    PUT /OS-EP-FILTER/projects/{project_id}/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_project``

Modifies the endpoint resource adding an association between the project and
the service provider.

Response:

::

    Status: 204 No Content

Check Association
^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/projects/{project_id}/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_project``

Verifies the existence of an association between a project and a service
provider.

Response:

::

    Status: 200 OK

List Associations for Project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/projects/{project_id}/service_providers

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_project``

Returns all the service providers ids that are currently associated with a
specific project.

Response:

::

    Status: 200 OK

    {
        "links": {
            "next": null,
            "previous": null,
            "self": "http://identity:5000/v3/OS-EP-FILTER/projects/abcd1234/service_providers"
        },
        "service_providers": [
            {
                "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
                "description": "Stores ACME identities",
                "enabled": true,
                "id": "ACME",
                "links": {
                    "self": "http://identity:5000/v3/OS-FEDERATION/service_providers/ACME"
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
                    "self": "http://identity:5000/v3/OS-FEDERATION/service_providers/ACME-contractors"
                },
                "relay_state_prefix": "ss:mem:",
                "sp_url": "https://other.example.com:5000/Shibboleth.sso/SAML2/ECP"
            }
        ]
    }


Delete Association
^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/projects/{project_id}/service_providers/{sp_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_project``

Eliminates a previously created association between a project and an service
provider.

Response:

::

    Status: 204 No Content

Get projects associated with service provider
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers/{sp_id}/projects

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_project``

Returns a list of projects that are currently associated with the given
service provider.

Response:

::

    Status: 200 OK

    {
        "links": {
            "self": "http://identity:5000/v3/OS-EP-FILTER/service_providers/6fedc0/projects",
            "previous": null,
            "next": null
        },
        "projects": [
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "263fd9",
                "links": {
                     "self": "http://identity:5000/v3/projects/263fd9"
                },
                "name": "a project name 1",
                "description": "a project description 1"
            },
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "61a1b7",
                "links": {
                     "self": "http://identity:5000/v3/projects/61a1b7"
                },
                "name": "a project name 2",
                "description": "a project description 2"
            }
        ]
    }


Service Provider Groups
~~~~~~~~~~~~~~~~~~~~~~~

Represents a group of ``service providers`` that should be accesible to users
with their tokens scoped to a specific project.

*New in version 1.2*

Required attributes:

- ``name`` (string)

User-facing name of the service.

- ``service_providers`` (list)

  A list of service provider IDs within a Service Provider Group.
  Note that only enabled service providers will be taken into consideration.

Optional attributes:

- ``description`` (string)

  User-facing description of the service.

Create Service Provider Group Filter
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    POST /OS-EP-FILTER/service_providers_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Request:

::

    {
        "service_provider_group": {
            "description": "Service Providers group description",
            "service_providers": [
                "sp1",
                "sp2",
                "sp3"
            ]
            "name": "endpoint group name",
        }
    }

Response:

::

    Status: 201 Created

    {
        "service_provider_group": {
            "description": "Service Providers group description",
            "id": "ac4861",
            "links": {
                "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/ac4861"
            },
            "name": "endpoint group name",
            "service_providers": [
                "sp1",
                "sp2",
                "sp3"
            ]
        }
    }

Get Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "service_provider_group": {
            "description": "Service Providers group description",
            "id": "ac4861",
            "links": {
                "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/ac4861"
            },
            "name": "endpoint group name",
            "service_providers": [
                "sp1",
                "sp2",
                "sp3"
            ]
        }
    }

Check Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

Update Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    PATCH /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``


Request:

::

    {
        "service_provider_group": {
            "description": "Service providers group description",
            "service_providers": [
                "sp1",
                "sp3"
            ]
            "name": "Service Provider group name",
        }
    }

Response:

::

    Status: 200 OK

    {
        "service_provider_group": {
            "description": "Service providers group description",
            "id": "ac4861",
            "links": {
                "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/ac4861"
            },
            "name": "Service Provider group name",
            "service_providers": [
                "sp1",
                "sp2",
                "sp3"
            ]
        }
    }

Remove Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 204 No Content

List All Service Provider Groups
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "links": {
            "self": "https://identity:5000/v3/OS-EP-FILTER/service_providers_groups",
            "previous": null,
            "next": null
        },
        "service_providers_groups": [
            {
                "service_provider_group": {
                    "description": "Service Provider group description #1",
                    "service_providers": [
                        "sp1",
                        "sp2",
                        "sp3"
                    ]
                    "id": "ac4861",
                    "links": {
                        "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/ac4861"
                    },
                    "name": "Service Provider group name #1"
                }
            }
            {
                "service_provider_group": {
                    "description": "Service Provider group description #1",
                    "service_providers": [
                        "sp10",
                        "sp22",
                        "sp33"
                    ]
                    "id": "cd4543",
                    "links": {
                        "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/cd4543"
                    },
                    "name": "Service Provider group name #2"
                }
            }
        ]
    }

List Service Provider Groups Associated with Project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/projects/{project_id}/service_providers_groups

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "links": {
            "self": "https://identity:5000/v3/OS-EP-FILTER/service_providers_groups",
            "previous": null,
            "next": null
        },
        "service_providers_groups": [
            {
                "service_provider_group": {
                    "description": "Service Provider group description #1",
                    "service_providers": [
                        "sp1",
                        "sp2",
                        "s[3"
                    ]
                    "id": "ac4861",
                    "links": {
                        "self": "http://localhost:5000/v3/OS-EP-FILTER/service_providers_groups/ac4861"
                    },
                    "name": "endpoint group name #1"
                }
            }
        ]
    }

Project to Service Provider Group Relationship
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create Service Provider Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    PUT /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 204 No Content

Get Service Provider Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "project": {
            "domain_id": "1789d1",
            "enabled": true,
            "id": "263fd9",
            "links": {
                "self": "http://identity:5000/v3/projects/263fd9"
            },
            "name": "project name #1",
            "description": "project description #1"
        }
    }

Check Service Provider Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    HEAD /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

Delete Service Provider Group to Project Association
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    DELETE /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects/{project_id}

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 204 No Content

List Projects Associated with Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "projects": [
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "263fd9",
                "links": {
                     "self": "http://identity:5000/v3/projects/263fd9"
                },
                "name": "a project name 1",
                "description": "a project description 1"
            },
            {
                "domain_id": "1789d1",
                "enabled": true,
                "id": "61a1b7",
                "links": {
                     "self": "http://identity:5000/v3/projects/61a1b7"
                },
                "name": "a project name 2",
                "description": "a project description 2"
            }
        ],
        "links": {
            "self": "http://identity:5000/v3/OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/projects",
            "previous": null,
            "next": null
        }
    }

List Service Service Providers Associated with Service Provider Group
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::

    GET /OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/service_providers

Relationship:
``http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/service_providers_groups_projects``

Response:

::

    Status: 200 OK

    {
        "links": {
            "self": "http://identity:5000/v3/OS-EP-FILTER/service_providers_groups/{service_providers_group_id}/service_providers",
            "previous": null,
            "next": null
        },
        "service_providers": [
            {
                "auth_url": "https://example.com:5000/v3/OS-FEDERATION/identity_providers/acme/protocols/saml2/auth",
                "description": "Stores ACME identities",
                "enabled": true,
                "id": "ACME",
                "links": {
                    "self": "http://identity:5000/v3/OS-FEDERATION/service_providers/ACME"
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
                    "self": "http://identity:5000/v3/OS-FEDERATION/service_providers/ACME-contractors"
                },
                "relay_state_prefix": "ss:mem:",
                "sp_url": "https://other.example.com:5000/Shibboleth.sso/SAML2/ECP"
            }
        ]
    }

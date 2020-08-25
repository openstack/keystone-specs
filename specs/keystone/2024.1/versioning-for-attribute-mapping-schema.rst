..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
Add schema version and support for the domain attribute in mapping rules
=========================================================================
This spec introduces support for the "domain" attribute in mapping rules,
and adds a versioning method to the attribute mapping schema used by identity
federation deployments.

`bug #1887515 <https://bugs.launchpad.net/keystone/+bug/1887515>`_

This specification implements an alternative solution for the versioned
mappings that were proposed in [1]_.

Problem Description
===================
Currently, Keystone identity provider (IdP) attribute mapping schema only
uses the "domain" attribute mapping as a default configuration for the domain
of groups being mapped; groups can override the default attribute mapping
domain by setting their specific domain. However, there are other "elements"
such as user and project that can also have a domain to define their location
in OpenStack.

An operator when reading the attribute mapping section and seeing the schema
for the attribute mapping definition, can be led to think that the domain
defined in the mapping will also apply to users and projects. However, that is
not what happens.

The examples in the documentation [2]_ do not cover all use cases.
Therefore, some people might go directly to the code, and check the schema
definition to understand what could be defined as the mapping. Then, the
operator might see that at the root of the `local` rules, we have a domain
definition [3]_. That options is also available for the `user` and `group`
properties. Therefore, one might think that those could be used as an
override, and this one ([3]_) is the default for the mapping rule. However,
that is not how the code is implemented.

Proposed Change
===============
First of all, to facilitate the development and extension concerning attribute
mappings for IdPs, we changed the way the attribute mapping schema is handled.
We introduced a new option called `schema_version`, which defaults to "1.0".
This attribute mapping schema version will then be used to control the
validation of the attribute mapping, and also the rule processors used to
process the attributes that come from the IdP. So far, with this patch, we
introduce support to the attribute mapping schema "2.0", which enables
operators to also define a domain for the projects they want to assign users.
If no domain is defined either in the project or in the global domain
definition for the attribute mapping, we take the IdP domain as the default.

Thus we extended Keystone identity provider (IdP) attribute mapping
schema to make Keystone honor the `domain` configuration that we have on it.
Currently, that configuration is only used to define a default domain for
groups (and then each group there, could override it). It is of interest to
expand this configuration (as long as it is in the root of the attribute
mapping) to be also applied for users and projects.

Attribute mapping schema
------------------------
The current schema defined at `keystone/federation/utils.py` does not have
a property to hold the schema version for the rules being defined. Therefore,
we will also introduce a new property there, called `schema_version`. This
new property will be a String that holds the mapping rule schema version.

The mapping rule schema version is then used to define the processor that we
take to process the attribute mapping rules JSON.


Before the proposed changes, one could create a mapping as follows (the
following mapping is only a sample; there are many other possibilities):

.. code-block:: python

  {
    "mapping": {
      "rules": [
        {
          "remote": [
            {
              "type": "OIDC-preferred_username"
            },
            {
              "type": "OIDC-email"
            },
            {
              "type": "OIDC-openstack-user-domain"
            },
            {
              "type": "OIDC-openstack-project-name"
            }
          ],
          "local": [
            {
              "domain": {
                "name": "{2}"
              },
              "user": {
                "domain": {
                  "name": "{2}"
                },
                "type": "ephemeral",
                "email": "{1}",
                "name": "{0}"
              },
              "projects": [
                {
                "domain": {
                  "name": "{2}"
                  },
                  "name": "{3}",
                  "roles": [
                    {
                      "name": "member"
                    }
                  ]
                }
              ]
            }
          ]
        }
      ],
      "links": {
        "self": "http://<keystone_server>/v3/OS-FEDERATION/mappings/<attribute_mapping_id>"
      },
      "id": "<attribute_mapping_id>",
    }
  }

With the proposed changes, one can for instance, use the following attribute
mapping. We are now able to only declare the domain at the root of the
mapping, and then it is re-used in all other elements that have it. Moreover,
it is possible to override it for specific objects.

.. code-block:: python

  {
    "mapping": {
      "rules": [
        {
          "remote": [
            {
              "type": "OIDC-preferred_username"
            },
            {
              "type": "OIDC-email"
            },
            {
              "type": "OIDC-openstack-user-domain"
            },
            {
              "type": "OIDC-openstack-extra-project-domain"
            },
            {
              "type": "OIDC-openstack-project-name"
            },
            {
              "type": "OIDC-openstack-extra-project-name"
            }
          ],
          "local": [
            {
              "domain": {
                "name": "{2}"
              },
              "user": {
                "type": "ephemeral",
                "email": "{1}",
                "name": "{0}"
              },
              "projects": [
                {
                  "name": "{4}",
                  "roles": [
                    {
                      "name": "member"
                    }
                  ]
                },
                {
                  "domain": {
                    "name": "{3}"
                    },
                  "name": "{5}",
                  "roles": [
                      {
                        "name": "member"
                      }
                    ]
                  }
                ]
              }
            ]
          }
        ],
        "links": {
          "self": "http://<keystone_server>/v3/OS-FEDERATION/mappings/<attribute_mapping_id>"
        },
        "id": "<attribute_mapping_id>",
        "schema_version": "2.0"
      }
    }

Validations
-----------
We will validate the `schema_version` against all existing/possible versions
of the field. Therefore, if it is not within the already defined version, an
error is thrown.

If no `schema_version` is presented, then the version used is `1.0`

Database table changes
----------------------

Currently, the table "mapping" is defined as:

.. code-block:: bash

    +-------+-------------+------+-----+---------+-------+
    | Field | Type        | Null | Key | Default | Extra |
    +-------+-------------+------+-----+---------+-------+
    | id    | varchar(64) | NO   | PRI | NULL    |       |
    | rules | text        | NO   |     | NULL    |       |
    +-------+-------------+------+-----+---------+-------+

We would add a newfield to it. Therefore, it would look like:

.. code-block:: bash

    +----------------+-------------+------+-----+----------+-------+
    | Field          | Type        | Null | Key | Default  | Extra |
    +----------------+-------------+------+-----+----------+-------+
    | id             | varchar(64) | NO   | PRI | NULL     |       |
    | rules          | text        | NO   |     | NULL     |       |
    | schema_version | varchar(5)  | NO   |     | '1.0'    |       |
    +----------------+-------------+------+-----+----------+-------+

API impacts
-----------
- a new attribute will be available `schema_version`. It defaults to `1.0`.
- Properties `groups`, `projects`, and `user`, all accept the definition of a
  `domain` that overrides the optional `domain` that is defined in the root of
  the mapping rule. If these objects do not define a domain, we take the one
  specified in the root of the mapping rule. If no domain is also defined in
  the root of the mapping rule, the IdP domain is used.


Assignee(s)
-----------

Primary assignees:
 - Rafael <rafael@apache.org>

Other contributors:

Work Items
----------

1) Implement proposed changes in Keystone [4]_

 - Create a new mapping schema

 - Create new processors for the proposed changes

 - Implement validations and unit tests

 - Update documentation

Dependencies
============

None

References
==========

.. [1] https://github.com/openstack/keystone-specs/blob/dfc7931a7f8975a98fc49fc3134cb923e31a5a76/specs/keystone/backlog/versioned-mappings.rst
.. [2] https://docs.openstack.org/keystone/latest/admin/federation/mapping_combinations.html
.. [3] https://github.com/openstack/keystone/blob/b0b93c03986f3bb40c5a2ec31ee37c83014e197a/keystone/federation/utils.py#L118
.. [4] https://review.opendev.org/#/c/739966/

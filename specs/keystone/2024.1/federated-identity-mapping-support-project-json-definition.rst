..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================================
Keystone identity mapping to support project definition as a JSON
=================================================================
This spec adds support for project definition as a JSON
in Keystone identity federation attribute mapping schema

Problem Description
===================
Currently, the project assignment via the federated identity mapping is rather
static. This happens because of the find/replace mechanism that we have in
place in Keystone. Therefore, if the IdP generates an attribute that
contains a JSON with project definitions, we are not able to handle it
in Keystone.

For instance, if we have the following attribute mapping, we are able to map
users to a single project in the OpenStack backend.

.. code-block:: python

    [
      {
        "local": [
          {
            "user": {
              "name": "{0}",
              "email": "{1}",
              "domain": {
                "name": "{2}"
              }
            },
            "domain": {
              "name": "{2}"
            },
            "projects": [
              {
                "name": "{3}",
                "roles": [
                  {
                    "name": "member"
                  }
                ]
              }
            ]
          }
        ],
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
            "type": "OIDC-openstack-default-project"
          }
        ]
      }
    ]

As mentioned, we would be able to map a user to a single project.
This limitation has to do with the way these mappings are processed.
If we wanted to add the user to two projects, then we need to
replicate the project definition twice, and all users would be added to two
projects at a time. If there is an unequal number of projects for users, then
this does not work. Unless, if we create mapping rules to address all possible
number of project assignments, which is infinite.

Proposed Change
===============
This PR introduces a new behavior in the federated identity mapping schema
for property `projects`. This property will accept a JSON string, that
defines all of the projects and their specific roles that the
user must receive when logging in to the OpenStack platform. Moreover, when
using this extension, roles (assigned to projects) are added and removed
on the fly.

The property `projects` will be able to either receive a string or a Python
object such as it does now. If the content is a Json encoded string, it
will be validated against the current `projects` object schema. That means,
it has to be a valid project list structure.

**Role assignments will be persistent to facilitate audit/troubleshooting
after the user accesses the system. Therefore, for every federated login,
the information Keystone holds regarding the user is updated; similarly
to what happens today with username, email, and so on.**

The extension is quite straight forward. We created a new ``schema_version``
(3.0). This new version enables the handling of `projects`  as Json encoded
strings in the attribute mapping.

Furthermore, we added code to handle the addition of extra roles for projects
and removal of roles that are present in OpenStack, but are not in the IdP
data. **This is a mechanism to make the state of the OpenStack federated user
consistent with the Identity provider user attributes.**

Attribute mapping schema
------------------------
By supporting ``projects`` as a ``string``, we enable operators to use
attributes in the IdP, that are a Json strings which define the projects where
the users must be placed in. Moreover, This JSON is then validated against the
project definition. The new option will be handled in version ``3.0`` of the
attribute mapping schema.

An example on how the Json content of the ``projects`` variable looks like is
presented as follows.

.. code-block:: python

 "[
    {\"name\":\"projectACME\",\"roles\":[{\"name\":\"member\"}],
        \"domain\":{\"name\":\"domainXYZ\"}
    },
    {\"name\":\"projectInDefaultDomain\",\"roles\":[{\"name\":\"member\"}]},
    {\"name\":\"otherProject\",\"roles\":[{\"name\":\"otherRole\"}],
        \"domain\":{\"name\":\"otherDomain\"}
    }
 ]"

A possible mapping rule with the new attribute would look like the following.
One must bear in mind that this is an example, and should not be used directly
in any production environment. When creating the attribute mapping rule, one
should use the configuration that best suits his/her needs.

.. code-block:: python

    [
       {
          "local":[
             {
                "user":{
                   "name":"{0}",
                   "email":"{1}",
                   "domain":{
                      "name":"{2}"
                   }
                },
                "domain":{
                   "name":"{2}"
                },
                "projects":"{3}"
             }
          ],
          "remote":[
             {
                "type":"OIDC-preferred_username"
             },
             {
                "type":"OIDC-email"
             },
             {
                "type":"OIDC-openstack-user-domain"
             },
             {
                "type":"OIDC-openstack-projects-client-mapper"
             }
          ]
       }
    ]


Database table changes
----------------------
None

API impacts
-----------
A new input type option for `projects` in the attribute mapping is available.
This new option would only be used when `schema_version` is set to `3.0`.


Assignee(s)
-----------

Primary assignees:
 - Rafael <rafael@apache.org>

Other contributors:

Work Items
----------

1) Implement proposed changes in Keystone [1]_

 - Create a new mapping schema

 - Create new processors for the proposed changes

 - Implement validations and unit tests

 - Update documentation

Dependencies
============

None

References
==========

.. [1] https://review.opendev.org/#/c/742235

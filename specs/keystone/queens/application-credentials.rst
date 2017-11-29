..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Application Credentials
=======================

`bp application-credentials <https://blueprints.launchpad.net/keystone/+spec/application-credentials>`_

Implement application-specific credentials as a new credential type in order to
improve security by limiting access and not embedding User credentials in
configuration files.

Problem Description
===================

Currently, automated API interactions -- such as service-to-service
communication, deployment tooling, and cloud-native applications -- must
authenticate with keystone as keystone Users in order to be granted access to
the resources they need to automate. This model, while convenient for keystone,
increases the risk of account compromise by requiring the distribution of
unencrypted passwords. It also exacerbates the potential damage an account
compromise can have since an application using a User's credentials has, as a
result, all the User's permissions. The model hinders mitigation of a
compromise because changing the compromised password necessitates downtime for
distributed applications using the password. The following use cases
demonstrate the situation faced by operators and end users today and the need
for an alternate model for authentication of applications.

.. note:: To avoid confusion, in this document User will be used to refer
          to the Keystone User Resource, Consumer will be used to refer
          to a Human who is consuming resources from an OpenStack Cloud,
          Deployer to a Human who manages an OpenStack Deployment and
          Application as non-Human consumer of OpenStack APIs. While the
          word "application" may carry connotations, it should be understood
          here to mean any amount of code or automation expected to interact
          with the OpenStack APIs in the absence of direct human interaction.

Use Cases
---------

* As an OpenStack Consumer, I want to create Applications that programmatically
  request a token and make API calls for resources in my project.

To accomplish this, one will typically store User credentials in a
configuration file, such as `clouds.yaml`_ so that scripts can access the
OpenStack API without Human interaction. Having User credentials that are
potentially used for other systems, such as with federated identity, in
configuration files and/or embedding them into a virtual machine, creates a
potential security risk.

* As an OpenStack Consumer, I want to limit the actions that an Application
  can make to a pre-defined subset of what my User otherwise has access to do.

By using a User's credentials, the Application will have all of that User's
access. This is likely more access than needed. For instance, today a regular
Consumer cannot set up an application that can only upload images to Glance but
cannot create or delete servers in Nova because they would need to create a
role and custom policy.

* As an OpenStack Consumer and an OpenStack Deployer, I need to be able to
  gracefully update credentials used by Applications without downtime for the
  Applications.

The current system of Users and Passwords requires simultaneously changing the
Password and updating any Application configuration. For a Deployer this means
updating all of the config files for all of the OpenStack services at the same
time. During the period between the Password being changed and config being
updated, the Application will not be able to talk to the OpenStack API and will
be down. The Application Credential should allow for the creation of a new
credential, rolling the new credential out, then deleting the old credential
for credential rotation with no downtime.

* As an OpenStack Deployer, I need to configure some OpenStack Services to be
  able to make API calls to other OpenStack Services.

Information about Service Users are currently stored in configuration files.
For example, the nova Service User is currently in the configuration of every
nova-api, nova-compute and neutron configuration file. At run time, those
services use those User Credentials to perform service-to-service actions.

From a security perspective, it can be considered unfavorable to have User
credentials written to a file on disk, especially if the User has elevated
privileges on a Project. It should be possible to apply the principle of least
privilege (POLP) and limit access to the minimum level required to complete the
task. For a Consumer this means being able to choose to not use the full User
credentials for automation. For a Deployer this means being able to easily make
service-specific authorizations to go into the config files.

.. _clouds.yaml: https://docs.openstack.org/developer/os-client-config/#config-files

Proposed Change
===============

Implement a new restricted credential type (Application Credentials) to be used
by Applications. Application Credentials will be assigned the same roles the
Creating User has at creation time.

.. note:: This idea was originally going to be called "API Keys" because the
          idea is similar to other industry usage around generating a secret
          for applications to use with an API. This plan was discontinued
          because the term, while not well-defined in the industry, carries
          specific connotations about its usage and implementation that this
          specification does not match.  The name 'Application Credentials' is
          chosen instead to be a generic term describing vaguely what this is
          intended to be, which is identification information intended for use
          only by applications.

Application Credentials will have the following characteristics:

* Immutable.
* Allow for optionally setting limits, e.g. 5 Application Credentials per User
  or Project, to prevent abuse of the resource.
* Assigned the set of current roles the creating User has on the Project at
  creation time, or optionally a list of roles that is a subset of the creating
  User's roles on the Project.
* Secret exposed only once at creation time in the create API response.
* Limited ability to manipulate identity objects (see `Limitations Imposed`_)
* Support expiration.
* Will be deleted when the associated User is deleted.

Application Credentials will be treated as credentials and not authorization
tokens, as this fits within the keystone model and is consistent with others
APIs providing application authentication. It also avoids the security and
performance implications of creating a new token type that would potentially
never expire and have custom validation.

.. note:: In the future, it will be possible for a Consumer to further limit,
          at creation time, the operations the Application Credential can
          perform. The intent is that such a further limitation will be
          expressed at a granularity of REST calls. That functionality is not
          described or implemented here, but is required to fully meet the Use
          Case associated with pre-defined subset of User actions.

Application Credential Management
---------------------------------

Users can create, list, and delete Application Credentials for themselves. For
example, adding an Application Credential:

::

    POST /v3/users/{user_id}/application_credentials

.. code-block:: json

    {
        "application_credential": {
            "name": "backup",
            "description": "Backup job...",
            "expires_at": "2017-11-06T15:32:17.000000",
            "roles": [
                {"name": "Member"}
            ]
        }
    }

`name` must be unique among a User's application credentials, but `name` is
only guaranteed to be unique under that User. `name` may be useful for
Consumers who want human readable config files.

`description` is a long description for storing information about the purpose
of the Application Credential. It is mostly useful in reports or listings of
Application Credential.

`expires_at` is when the Application Credential expires. 'null' means that the
Application Credential does not automatically expire. `expires_at` is in `ISO
Date Time Format`_ and is assumed to be in UTC if an explicit timezone offset
is not included.

`roles` is an optional list of role names or ids that is a subset of the roles
the Creating User has on the Project to which they are scoped at creation time.
Roles that the Creating User does not have on the Project are an error.

In the initial implementation, the Application Credential will assume the roles
of the Creating User or the given subset and we will not implement
fine-grained access controls beyond that.

Response example:

.. code-block:: json

    {
        "application_credential": {
            "id": "aa4541d9-0bc0-44f5-b02d-a9d922df7cbd",
            "secret": "a49670c3c18b9e079b9cfaf51634f563dc8ae3070db2...",
            "name:" "backup",
            "description": "Backup job...",
            "expires_at": "2017-11-06T15:32:17.000000",
            "project_id": "1a6f968a-cebe-4265-9b36-f3ca2801296c",
            "roles": [
                {
                    "id": "d49d6689-b0fc-494a-abc6-e2e094131861",
                    "name": "Member"
                }
            ]
        }
    }

The `id` in the response is the Application Credential identifier and would be
returned in get or list API calls. An `id` is globally unique to the cloud.

`secret` is a random string and only returned via the create API call.
Keystone will only store a hash of the `secret` and not the `secret` itself,
so a lost `secret` is unrecoverable. Subsequent queries of an Application
Credential will not return the secret field.

.. note:: Identifying the correct way to generate an acceptable secret needs
          to be done. Nova generates admin passwords on server creation, which
          is probably a good place to start. If that approach is taken, use of
          `random.choice` should be replaced for Python 3 with
          `secrets.choice`.

`roles` is a list of role names and ids. It is informational and can be used by the
Consumer to verify that the Application Credential inherited the roles from
the User that the Consumer expected. This is not a policy enforcement, it is
simply for human validation.

If the Consumer prefers to generate their own `secret`, they can do so and
provide it in the create call. Keystone will store a hash of the given
`secret`. Keystone will return the secret once upon creation in the same way it
would if it was generated, but will not store the secret itself nor return it
after the initial creation.

A Consumer can list their existing Application Credentials:

::

    GET /v3/users/{user_id}/application_credentials

.. code-block:: json

    {
      "application_credentials": [
        {
            "id": "aa4541d9-0bc0-44f5-b02d-a9d922df7cbd",
            "name:" "backup",
            "description": "Backup job...",
            "expires_at": "2017-11-06T15:32:17.000000",
            "project_id": "1a6f968a-cebe-4265-9b36-f3ca2801296c",
            "roles": [
                {
                    "id": "d49d6689-b0fc-494a-abc6-e2e094131861",
                    "name": "Member"
                }
            ]
        }
      ]
    }

A Consumer can get information about a specific existing Application
Credential:

::

    GET /v3/users/{user_id}/application_credentials/{application_credential_id}

.. code-block:: json

    {
      "application_credentials": [
        {
            "id": "aa4541d9-0bc0-44f5-b02d-a9d922df7cbd",
            "name:" "backup",
            "description": "Backup job...",
            "expires_at": "2017-11-06T15:32:17.000000",
            "project_id": "1a6f968a-cebe-4265-9b36-f3ca2801296c",
            "roles": [
                {
                    "id": "d49d6689-b0fc-494a-abc6-e2e094131861",
                    "name": "Member"
                }
            ]
        }
      ]
    }

A Consumer can delete one of their own existing Application Credential to
invalidate it:

::

    DELETE /v3/users/{user_id}/application_credentials/{application_credential_id}

.. note:: Application Credentials that expire will be deleted. The alternative
          would be to allow them to accumulate for forever in the hopes that
          keeping them around will make investigation as to why an Application
          is not working easier, but the only real benefit to this is providing
          a different error message. More thought and feedback on this are
          needed, but are not essential for the first round of work.

When the Creating User for an Application Credential is deleted, or if their
roles on the Project to which the Application Credential is scoped are
unassigned, that Application Credential is also deleted.

Aside from deletion, Application Credentials are immutable and may not be
modified.

.. _ISO Date Time Format: https://tools.ietf.org/html/rfc3339#section-5.6

Using an Application Credential to Obtain a Token
-------------------------------------------------

An Application Credential can be used for authentication to request a scoped
token following Keystone's normal authorization flow. For example:

::

    POST /v3/auth/tokens

.. code-block:: json

    {
        "auth": {
            "identity": {
                "methods": [
                    "application_credential"
                ],
                "application_credential": {
                    "id": "aa4541d9-0bc0-44f5-b02d-a9d922df7cbd",
                    "secret": "a49670c3c18b9e079b9cfaf51634f563dc8ae3070db2..."
                }
            }
        }
    }

Keystone will validate the Application Credential by matching a hash of the key
secret associated with the id using `passlib`_ similar to how Keystone does
Password authentication currently.

If the Application Credential is referred to by `name`, it will be necessary to
provide either `user_id` or the combination of `user_name` and
`user_domain_name` so that Keystone can look up the Application Credential
for the User.

::

    POST /v3/auth/tokens

.. code-block:: json

    {
        "auth": {
            "identity": {
                "methods": [
                    "application_credential"
                ],
                "application_credential": {
                    "name": "backup",
                    "user": {
                        "id": "1a6f968a-cebe-4265-9b36-f3ca2801296c"
                    },
                    "secret": "a49670c3c18b9e079b9cfaf51634f563dc8ae3070db2..."
                }
            }
        }
    }

As an alternative to the current use of Service Users, a Deployer could
create a single Service User and an Application Credential for each service. Or
even create a Nova user and then give each nova instance it's own Application
Credential. Although at this point the Application Credential does not have
the ability to further limit API use, the ability to start assigning
Application Credentials per-service and performing expiration and rotation may
be a desirable step forward that can be further enhanced with the addition of
restricting an Application Credential's API Access.

.. _passlib: https://pypi.python.org/pypi/passlib

Limitations Imposed
-------------------

In general, Application Credentials should be prevented from themselves
generating other Application Credentials, because we do not want to allow a
compromised Application Credential to make copies of itself. By extension, they
should be prevented from creating trusts since a self-trust could also be used
to generate another Application Credential. Since API Access Lists are not
implemented at this stage, Keystone will, initially, by default, explicitly
block tokens generated from an Application Credential from doing the following:

* POST /v3/users/{}/application_credentials
* DELETE /v3/users/{}/application_credentials/{}
* POST /v3/OS-TRUST/trusts
* DELETE /v3/OS-TRUST/trusts/{}

It will do this by checking for an "application_credential" value in the
"methods" key of a token object.

This block can be disabled by adding an
``"allow_application_credential_creation": true`` property to the application
credential at creation. This is to support a use case specific to Heat, where
the Heat user's application credential should be allowed to create new
application credentials to be injected into an instance. Even though the
purpose is to accomodate this particular service, any user can effectively
declare "I accept the risks" and enable this potentially dangerous behavior by
adding this property at creation time.

In the future, more fine-grained API Access List Controls will remove the need
for such a block and the workaround for the block.

Design Justifications
---------------------

Implementing Application Credential management as normal CRUD with default
policy that normal Users have access to adds the ability without requiring the
Deployer to perform additional setup or manage additional external services.

Implementing Application Credentials consumption as an auth-type plugin means
that any Client code that supports pluggable auth in any way should be able to
easily consume the new feature. Client code that doesn't yet implement support
for pluggable authentication should have a compelling motivation to add it.

Alternatives
============

`Enhance tokens`_ by allowing delegation of subsets of roles to a token. This
solves the problem of granting too much access to an application, but it still
necessitates tying an application to a User, and token expiry makes it
insufficient for use by applications.

`Enhance users`_ by adding new credential types and then allowing role
assignments to be assigned to credential types instead of users. Regular
Consumers frequently do not have permission to create Users, especially in
places that use an identity backend like LDAP or AD.

Just use OAuth. Keystone already supports OAuth-based authentication. However,
adding OAuth on top of an existing Auth system is a deployer opt-in task that
involves considerable deployer effort. If the goal is to add support for a
concept that will always dependably exist, OAuth represents too high of a
burden to be reasonable.

`Enhance trusts`_ by detaching them from trustees. Trusts still require the
User's password for authentication, and so it prevents seamless rotation. This
implementation may build on the existing framework for trusts but must build in
a secret that is separate from the User's password to allow for that rotation.

On naming: GCE uses the term "Service Account" and discusses the concept of
"Application Access". Both could be useful alternate terms to Application
Credential. Drawbacks are that "Service Account" uses the word "account" which
may run afoul of understanding. "Application Access" is a good description of
what we wish to provide but "Access" does not well-denote the unique unit of
information that can be requested and submitted, where "Credential" does.

Github exposes a similar concept as `GitHub App`_ which are special accounts
intended to be used for non-human API access, such as bots or other automation.
They are similarly intended to be a project-level resource rather than a
user-level resource. "App" could be a name for this. However, the Github
concept goes hand-in-hand with a catalog of services that people can
register to do things on their accounts, which is well out of scope for this
proposal. Additionally the term "App" carries connotations that may not be
appropriate for people who would use this construct as part of a general
automation system. Using that term is likely to cause more confusion than good.

.. _`Enhance tokens`: https://review.openstack.org/#/c/186979
.. _`Enhance users`: https://review.openstack.org/#/c/389870
.. _`Enhance trusts`: https://review.openstack.org/#/c/396634
.. _`GitHub App`: https://developer.github.com/apps/

Security Impact
---------------

This would have a positive security impact:

* Instead of having a Service User for each service, all services can use a
  single Service User and multiple Application Credentials. This decreases the
  attack vector of gaining access to privileged operations by reducing the
  number of accounts to attack.
* Usernames and passwords are kept out of configuration files. While
  Application Credentials are still extremely sensitive, if compromised they do
  not allow attackers to glean service user password conventions from
  configuration.
* Application Credentials will grow the ability to have limited access, so a
  move to them is a step towards limited access credentials.
* Application Credentials can be gracefully rotated out of use and deleted
  periodically, allowing Consumers and Deployers a mechanism to prevent
  compromised Users without requiring swapping credentials in short amounts of
  time that might cause service interruption or downtime.
* Although we had long considered allowing application credentials to live
  beyond the lifetime of its creating user in order to allow seamless
  application uptime when the user leaves the team, it unfortunately poses too
  high a risk for abuse. Ensuring the application credential is deleted when
  the user is deleted or removed from the project will prevent malicious or
  lazy users from giving themselves access to a project when they should no
  longer have it.

There is an inherent risk with adding a new credential type and changing
authentication details. One such risk would be the allowing of many credentials
for the same User account.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

* Consumers who have Applications that monitor or interact with OpenStack
  Services should be able to leverage this feature to improve the overall
  security and manageability of their Applications.
* Consumers can gracefully rotate Application Credentials for an
  Application with no downtime by creating a new Application Credential,
  updating config files to use the new Application Credential, and finally
  deleting the old Application Credential.
* Consumers who do not start using Application Credentials should experience no
  impact.

Performance Impact
------------------

This should have the same performance impact as username/password
authentication does today, since the same mechanism will be used to compare
hashes to obtain an answer.

Other Deployer Impact
---------------------

Deployers should notice the following maintenance improvements:

* Deployers only need to enforce security on a single Service User instead
  of multiple.
* Password rotation policies for Service Users no longer require immediately
  redeploying service configuration files. A User password change does not
  affect the existing Application Credential in the various service
  configuration files.
* Deployers can gracefully rotate Application Credentials through a deployment
  with no downtime.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

- Monty Taylor

Other contributors:

- Colleen Murphy

Work Items
----------

#. Develop backend to support Application Credentials

#. Implement API, business logic, and validation for CRUD operations

#. Add Credential deletion to User deletion

#. Add new Application Credential authentication plugin

#. Block access to Keystone application credential APIs

#. Add consumption support to keystoneauth

#. Documentation

#. Notify deployment projects about using Application Credentials
   (Devstack, OpenStack-Ansible, Tripleo, OpenStack Puppet, etc)

#. Notify non-Python SDKs about using Application Credentials (Fog,
   gophercloud, JClouds)

Dependencies
============

None

Documentation Impact
====================

The in-tree installation guide should be updated to account for Application
Credentials and Service Users. The user guides should also be updated to
reflect the ability for Users to use Application Credentials for application
authentication.

References
==========

http://lists.openstack.org/pipermail/openstack-dev/2017-May/116596.html

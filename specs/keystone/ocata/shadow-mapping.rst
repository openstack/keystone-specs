..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
Shadow mapping
==============

`bp shadow-mapping <https://blueprints.launchpad.net/keystone/+spec/shadow-mapping>`_

At the Newton summit in Austin, several related questions about how deployers
solve real world problems with automation kept appearing in loosely related
conversations:

- How are federated users bootstrapped into the cloud with some amount of
  useful authorization?

- How do federated users receive role assignments?

- Federation or not, how do deployers create "personal projects" for new users?
  How are new users assigned default projects?

Today, deployers have to answer these questions by implementing external
orchestration tools which either preemptively populate keystone with
authorization data in bulk, or intervene during the authentication process to
do so only when necessary.

With the introduction of shadow users in the Mitaka release, Keystone is now
capable of persisting local identities to reflect remote users, thus
opening the door for locally-managed role assignments rather than just
ephemeral group memberships. The next logical step is to extend that capability
to handle the other pieces of the authorization puzzle: projects, roles, role
assignments, and default projects.

Problem Description
===================

A shadow user is created only after a successful authentication attempt.
Therefore, a federated user would have to attempt to authenticate before an
administrator could assign roles directly to their shadowed identity, resulting
in a strange user experience. Granted, we do have the capability to manage
authorization on user groups ahead of time, user groups are simply not a
flexible enough solution to cover everyone's use cases. For example, how do we
"map" new user into a personal project (a user-specific tenancy), assign the
user a "member" role on that project, and make it the user's default project
for subsequent authentication attempts?

Proposed Change
===============

The mapping engine is responsible for mapping federated attributes into local
ones. Today, it's essentially capable of determining a user's name and group
memberships. This change proposes to also "map" users into local projects and
roles, with local role assignments, which may or may not exist at the time of
authentication.

For example, if a project name is referenced in the mapping, it may need to be
automatically created prior to assigning the user a "member" role on it (where
the member role may or may not exist as well).

The following is an example of what the mapping engine supports today:

.. code:: javascript

    PUT /OS-FEDERATION/mappings/{mapping_id}
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

The following is an example of what the mapping engine might support as a
result of this spec:

.. code:: javascript

    PUT /OS-FEDERATION/mappings/{mapping_id}
    {
        "mapping": {
            "domain_id": "ab4e2e",
            "rules": [
                {
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
                    ],
                    "local": [
                        {
                            "user": {
                                "name": "{0}"
                            }
                        },
                        {
                            "projects": [
                                {
                                    "name": "Development project for {0}",
                                    "roles": [
                                        {
                                            "name": "admin"
                                        }
                                    ]
                                },
                                {
                                    "name": "Staging",
                                    "roles": [
                                        {
                                            "name": "member"
                                        }
                                    ]
                                },
                                {
                                    "name": "Production",
                                    "roles": [
                                        {
                                            "name": "observer"
                                        }
                                    ]
                                },
                            ]
                        }
                    ]
                }
            ]
        }
    }

The above example is constructed with the following considerations in mind:

- The mapping explicitly references a ``domain_id``, which applies to all
  objects in the mapping schema (users, projects, and possibly even roles).
  This would allow a domain administrator to control a mapping, without
  requiring intervention from the cloud operator.

- The mapping refers to multiple projects, each with a unique set of role
  references. This implies that the user has those role assignments on each of
  the respective projects.

- Each project name (and possibly ID) may be determined dynamically based on
  remote assertions. If any of those projects or roles do not exist, they must
  be created by Keystone automatically. Since the dynamic values come from the
  assertion, it is safe to assume they only need to be created once.

- The user's ``default_project_id`` attribute could be automatically set to the
  first project that appears in the list. This could also be something that is
  added at a later time. Initially it wouldn't be a requirement for a user's
  ``default_project_id`` to be set.

So, in this example, let's say that the remote ``UserName`` attribute is simply
"Joe". According to the mapping, if Joe is neither a guest nor contractor, he
would:

1. Receive a shadowed user identity, with a username of "Joe", in the domain
   with an ID of "ab4e2e".

2. Receive a project-scoped "admin" role on a new project (created
   automatically) named "Development project for Joe" in the "ab4e2e" domain.

3. Joe's ``default_project_id`` would be set to the ID of the "Development
   project for Joe".

4. Receive direct user + project + role assignments on three projects, with
   three different roles.

5. Receive a project-scoped token (instead of an unscoped token, as federated
   users receive today), scoped to the user's default project. This reflects
   the auth behavior of local keystone users.

Finally, it's worth mentioning that if the mapping were to change between
federated authentications with keystone, the result of the new mapping would
simply be applied without any additional side effects. Any reduction in
authorization implied by a change in mapping would need to be handled out of
band, as Keystone would have no way of tracking what authorization was granted
as a result of a mapping versus any other means. However, normal token
revocation behaviors would still apply to the role assignments created by a
mapping (so you could still change a mapping, delete a project created by a
mapping, and expect tokens to be revoked for that project).

Alternatives
------------

One alternative is to have external orchestration tools `ask keystone to
predict user identities <https://review.openstack.org/#/c/313604/>`_, and
preemptively populate Keystone with appropriate authorization data before the
user attempts to authenticate. This makes several assumptions and places
additional design constraints on Keystone:

1. The operator is assuming that the user will successfully authenticate at
   some point in the future, thus making the prepopulated authorization data
   relevant and useful.

2. The mapping must be defined as it was when the operator queried for the
   result of the mapping as when the user finally authenticates.

3. Shadow user IDs must ultimately be repeatable (either predictable or
   persistent) rather than just arbitrary UUIDs lazily assigned during
   authentication.

4. We must assume that operators are willing to (continue to) implement such
   external orchestration tools. This may be acceptable for large deployments,
   but is an impractical barrier for smaller ones.

Security Impact
---------------

The mapping engine already has a relatively high impact on keystone's security
model, as it is a relatively complex and essentially dynamic source of user
identity and authorization management. That complexity will only increase as
the mapping engine is extended to handle additional capabilities around
authorization management. Deployers should carefully consider their security
policy around the mapping API itself.

We may also need to consider implement additional constraints over what
what domains the mapping engine can interact with, what projects can be
created, what roles can be assigned, etc.

Notifications Impact
--------------------

This has the potential to cause a lot of notification traffic when users are
first authenticated, as a large number of resources may be allocated at once.
The same would be true if an external tool were to create the same resources,
however.

Other End User Impact
---------------------

First time users will have a much smoother experience going through the
federated authenticating flow, without requiring significant, external effort
on the part of operators.

Performance Impact
------------------

For new users requiring a large number of resources to be allocated during
their first authentication, performance of that call will certainly suffer, as
the resources will be created synchronously. Aggressive client-side timeouts
(for example, in Horizon) may result in false-positive authentication failures.

Other Deployer Impact
---------------------

Defining mapping rules will ultimately be far more complicated, but the
trade-off is that deployers will not have to manage custom tooling on top of
keystone.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

- Ron De Rose (rderose)
- Lance Bragstad (lbragstad)

Work Items
----------

1. Extend the mapping engine's JSON schema to support projects, role
   assignments, and default projects.

2. Handle the additional output of the mapping engine to create and assign
   resources as required.

3. Thoroughly document the behavior of the mapping engine with existing mapping
   engine documentation.

Dependencies
============

None.

Documentation Impact
====================

The additional complexity of the mapping engine will require significant effort
to comprehensively document.

References
==========

- Keystone's `work session etherpad
  <https://etherpad.openstack.org/p/newton-keystone-work-session>`_ from the
  Newton summit in Austin.

- `"Federated user experience + shadow users" etherpad
  <https://etherpad.openstack.org/p/keystone-newton-mapping-engine>`_ from the
  Newton summit in Austin.

- Alexander Makarov's `Federation user story
  <http://lists.openstack.org/pipermail/openstack-dev/2016-May/095810.html>`_
  thread on the openstack-dev mailing list.

- Adam Young's `"Federated query APIs" specification
  <https://review.openstack.org/#/c/313604/>`_.

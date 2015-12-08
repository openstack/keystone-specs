..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
Shadow users: Unified identity for multiple authentication sources
==================================================================

`bp shadow-users <https://blueprints.launchpad.net/keystone/+spec/shadow-users>`_

Locally managed users are handled slightly differently than users backed by
LDAP, which are handled significantly differently than users backed by
federation. Available APIs, relevant APIs, and token validation responses all
vary. For example, users receive different types of IDs, passwords may or may
not be stored in keystone, and in the case of federation, may not be
able to receive direct role assignments. Future additional authentication
methods pose a risk of complicating things further.

Instead of continuing down this path, we can refactor our user persistence to
separate identities from their locally-managed credentials, if any. The result
will be a unified experience for both end users and operators.

Problem Description
===================

Use cases:

- **A consistent user experience.** A federated user should have the same user
  experience as a locally-managed user. For example, federated users should be
  able to consume local role assignments just like locally-managed users can.
  Today, federated users can only be mapped into user groups, and receive
  tokens with larger payloads to accommodate their ephemeral nature.

- **One user, multiple credentials.** In the real world, a single person has
  multiple means of authenticating themselves. For example, a person might
  carry a driver's license, have a passport, and know a secret, such as a
  credit card verification value (CVV). In terms of OpenStack, a user could
  have a password in keystone, a private identity provider (such as a
  corporate LDAP), a public identity provider (such as a social media profile),
  an X.509 certificate, and an existing account on a remote cloud. If all of
  these authentication sources are equally valid, then the resulting user and
  operator experience should not vary based on which is chosen. All of these
  means of authentications should be tied to the same user identity, rather
  than resulting in distinct identities.

  Note: We discussed the potential future need to tie privileges to
  authentication methods. This is out-of-scope for this spec, however the spec
  does not prevent or conflict from this feature being added in the future.

- **Additionally: Facilitates multi-factor authentication & account linking.**
  With multiple types of credentials tied to the same user identity, users can
  authenticate themselves using more than one credential at a time. Keystone
  can then make stronger assertions about the identity of the user and the path
  to viable multi-factor authentication (MFA) is shortened. Likewise, the
  proposed changes support linking multiple accounts for the same user to a
  single account, as well as simplifies auditing around federated users. While
  this specification does not solve MFA or account linking, the refactoring
  done for this spec will make the development for those features easier.

Proposed Change
===============

1. **Separate user identities from their local-managed credentials.** Refactor
   `user` table into an `identity` table and a locally-managed `password`
   table. Migrate data from the user table to these new tables and ultimately
   remove the user table. Modify backend code to utilize the new tables.

2. **Shadow LDAP and federated users.** Create new shadow tables for mapping
   LDAP and federated users to local identities. Federated users have a
   `idp_id`, `protocol_id`, display name, and a unique ID asserted by the
   identity provider. These are the minimal pieces of data required to identify
   returning users and provide them with a consistent `identity`. Likewise,
   LDAP users have a 'domain_id' and 'dn' used to identify the user. That being
   said, there may be an opportunity to generalize LDAP and federated users
   a single table. This will be solved during development.

Alternatives
------------

We can continue treating federated users as ephemeral. In the long run, that's
either going to result in additional metadata being included in token payloads,
ultimately bloating Fernet tokens beyond their intended capacity, or
increasingly disparate user experiences depending on the user's authentication
source.

Security Impact
---------------

* User IDs may be used as a source of authorization in OpenStack, and this spec
  will impact how they are controlled. We need to ensure that two unique users
  cannot be accidentally mapped to the same identity. The simplest solution to
  this is to always assign random UUIDs to serve as the OpenStack-facing user
  identity. We'll have to carefully consider the implications of using any
  other source.

* We'll gain the ability to remove information about federated users from
  tokens, thus eliminating the concept of "federated tokens" altogether.

* Federated user group membership will become persistent rather than ephemeral
  per authentication. A federated user might automatically receive a concrete
  role assignment as the result of the federation mapping and receive a Fernet
  token which reflects those roles. During the lifetime of that token, an
  operator might assign or revoke additional roles which will be immediately
  reflected in the user's existing token. This behavior is already the case for
  locally-managed users, but will be new to federated users.

Notifications Impact
--------------------

Auditing notifications which made use of federated identity information in
tokens will no longer have access to that information, beyond multiple
authentication methods presented by v3 tokens. All auditing notifications
will effectively refer to a local user identity.

Other End User Impact
---------------------

All end users will receive a consistent API experience: that of a
locally-managed one.

Performance Impact
------------------

* All user identities will need to be shadowed in keystone's local backend.
  This means that if you have millions of users authenticating with keystone,
  even if it's via identity federation, each of them will have a record in
  keystone.

* The code path for token validations will be simplified, because they will
  always refer to local users.

* Token formats requiring persistence will require less space on disk or in
  memory.

* Federated users will be able to consume local role assignments, slightly
  increasing the response time of token validation.

Other Deployer Impact
---------------------

* No new configuration options.

* Local identity persistence will need to be provided, even for deployments
  exclusively using identity federation.

* During the upgrade, a database migration will be required to split
  locally-managed users. At runtime, shadow records will be created
  automatically as LDAP and federated users successfully authenticate with
  keystone.

Developer Impact
----------------

Developers supporting alternative authentication methods will be able to
reference the common user identity, and create shadow records themselves. It's
important to note that existing CRUD APIs will not be impacted; only the
backend is being refactored.

With federation being a more viable option, deployers supporting custom code to
bootstrap new users with role assignments can take advantage of federated
mappings to accomplish the same thing, if they move to federation.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dolph

Other contributors:
  dstanek
  derosenet

Work Items
----------

1. Separate user identities from their local-managed credentials.

2. Shadow federated and LDAP users in new database tables.

3. Make concrete role assignments after federated mapping is evaluated.

4. Notifications emitted when a federated users is authenticated will need to
   be updated.

5. The mapping engine and other related backend logic will likely need some
   refactoring.

Dependencies
============

None.

Documentation Impact
====================

Deployers need to understand the new local user persistence requirements, even
in the case of federation.

Documentation that suggests that federated users cannot receive local role
assignments needs to be revised.

References
==========

* Etherpad notes from the Mitaka summit `federation session
  <https://etherpad.openstack.org/p/keystone-mitaka-summit-federation>`_.

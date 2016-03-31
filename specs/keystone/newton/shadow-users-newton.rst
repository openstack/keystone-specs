..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================================================
Shadow users (Newton): Unified identity for multiple authentication sources
===========================================================================

`bp shadow-users-newton <https://blueprints.launchpad.net/keystone/+spec/shadow-users-newton>`_

.. NOTE::

    This is a continuation of the remaining work that began in the `Mitaka
    release
    <http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html>`_.
    Only the summary is reproduced here for convenience.

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

See the `Mitaka spec (Problem Description)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#problem-description>`_.

Proposed Change
===============

See the `Mitaka spec (Proposed Change)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#proposed-change>`_
for the originally-proposed changes and additional detail.

#. **Separate user identities from their local-managed credentials.** *This was
   accomplished in the Mitaka release.*

#. **Shadow federated users.** *This was accomplished in the Mitaka release.*

#. **Create concrete role assignments for federated users.**

#. **Shadow LDAP users.**

#. **Drop the "EPHEMERAL" user type mapping.** Given that federated users are
   no longer ephemeral, we can ignore the "ephemeral" vs "local" user type and
   treat all users equally.

#. **Relax the requirement for mappings to result in group memberships.** Now
   that we're able to grant authorization to federated users using concrete
   role assignments, we can drop the requirement for the mapping engine to
   result in any authorization (via group membership) at all.

Alternatives
------------

See the `Mitaka spec (Alternatives)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#alternatives>`_.

Security Impact
---------------

See the `Mitaka spec (Security Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#security-impact>`_.

Notifications Impact
--------------------

See the `Mitaka spec (Notifications Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#notifications-impact>`_.

Other End User Impact
---------------------

See the `Mitaka spec (Other End User Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#other-end-user-impact>`_.

Performance Impact
------------------

See the `Mitaka spec (Performance Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#performance-impact>`_.

Other Deployer Impact
---------------------

See the `Mitaka spec (Other Deployer Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#other-deployer-impact>`_.

Developer Impact
----------------

See the `Mitaka spec (Developer Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#developer-impact>`_.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* rderose

Other contributors:

* dstanek
* dolphm

Work Items
----------

#. Shadow LDAP users in new database tables.

#. Make concrete role assignments after federated mapping is evaluated.

#. Update federated authentication notifications to reflect local identities.

#. Refactor the mapping engine (to resolve tech debt).

#. Drop support for "federated" tokens.

Dependencies
============

See the `Mitaka spec (Dependencies)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#dependencies>`_.

Documentation Impact
====================

See the `Mitaka spec (Documentation Impact)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#documentation-impact>`_.

References
==========

See the `Mitaka spec (References)
<http://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html#references>`_.

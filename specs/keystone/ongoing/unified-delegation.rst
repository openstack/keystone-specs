..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Unified model for assignments, OAuth and trusts
===============================================

`bp unified-delegation <https://blueprints.launchpad.net/keystone/+spec/unified-delegation>`_

Merge keystone delegation models to a single one with their capabilities
merged.

Problem Description
===================

Roles Assignments, OAuth tokens, and trusts all serve one single purpose:
to delegate roles on the *resource* to the *actor*.
*Resource* may be either project or domain, *actor* is a user or a group.

The current architecture does not maintain a chain of responsibility for
tracking what user originally created the role assignment, nor does it have
any means to restrict its usage. The trusts is more a workaround
rather than the sole solution for its own use case.

A scoped token represents a short term delegation for performing the
set of operations necessary to complete a single workflow.  Access
control is performed by matching token contents against policy
criteria and by validation of the token itself via keystone call.


Proposed Change
===============

Unify Assignment, OAuth RequestToken and Trusts into a single, unified
delegation model containing following information:

  - trustor: *actor* who delegates authorization scope
  - trustee: *actor* receiving a subset of trustor's powers
  - sources: model-specific fields for ancestry chain tracking, maintained
    internally

    * sequence of delegations preceding the current one in the ancestry chain
    * sequence of *agents* that created every delegation in the ancestry chain

      *Agent is the user who actually created the delegation - not necessary
      the trustor of the parent one*

  - strict_ancestry: flag, indicating whether to check that all agents and
    delegations in the ancestry chains are enabled
  - sealed: flag, prohibiting the possibility to create derivative delegation
  - executable: flag, indicating that *actor* is allowed to perform any
    delegated role - not only delegate it to the next *actor*, if *sealed*
    flag is set to False
  - roles to delegate
  - target: the *resource* (project or domain) the delegation is scoped to
  - expires (optional): time limiting the validity of tokens issued based on
    this delegation
  - remaining uses (optional): limits the number of tokens to be issued based
    on this delegation


Delegation will track the responsibility chain so that any delegation
is always granted by some actor to another. To allow this keystone
must maintain chain consistency: it must handle the cases where the
chain is broken or changed.

Delegation revocation must be cascaded down.

A delegation must have an option to restrict it's usage so that it can be used
for defined workflow and nothing more.

**Delegation model**

+--------------------+-------------------------------------------------------+
| Field              | Meaning/example                                       |
+====================+=======================================================+
| identifier         | [uuid]                                                |
+--------------------+-------------------------------------------------------+
| user_chain         | trustor/agent/agent/.../*actor(trustee)*              |
+--------------------+-------------------------------------------------------+
| delegation_chain   | delegation/.../parent_delegation/**this_delegation**  |
+--------------------+-------------------------------------------------------+
| strict_ancestry    | "check, that everything **user_chain** and            |
|                    | **delegation_chain** are referencing, enabled"        |
+--------------------+-------------------------------------------------------+
| sealed             | "do not allow further delegation"                     |
+--------------------+-------------------------------------------------------+
| executable         | "can issue a token scoped to this delegation"         |
+--------------------+-------------------------------------------------------+
| target             | [*resource(domain or project)*]                       |
+--------------------+-------------------------------------------------------+
| expires            | 2020/12/31T23:59:59                                   |
+--------------------+-------------------------------------------------------+
| remaining_uses     | Amount of tokens yet to be allowed to be issued       |
+--------------------+-------------------------------------------------------+

**Delegation roles**

+-------------------+
| Role m2m          |
+===================+
| delegation_id     |
+-------------------+
| role_id           |
+-------------------+

When implied roles feature is landed it should be possible to delegate a scope
holding the role considered implied to that specified in this m2m table.
In case if role is domain specific this is to be respected automatically as
the scope of the delegation is already restricted to the domain.

Alternatives
------------

Leave everything as it is - it still works and still can be used.

Security Impact
---------------

* Assignment, trust and OAuth1 request token storage is changed
* New delegation API is introduced
* Delegation search uses HTTP query parameters via hints
* Policy should be changed to allow regular user to create a delegation using
  existing assignment API
* Token content may change from the actual scope to mere reference to the
  delegation it is scoped to

Notifications Impact
--------------------

Notifications will be sent as for assignment operations now:

* created
* deleted
* disabled
* updated

Other End User Impact
---------------------

Unified delegation has the following side effect: delegations created using
the trust API, for example, can be accessed through the assignment API.

Exposing unified delegation through its own API should be followed by
corresponding extension of python-keystoneclient library. This is not
currently in scope.

Performance Impact
------------------

Delegation logic doesn't differ very much from that of assignment and trust,
though performance tests must be done, as delegation should be used for token
issuing and validation.

Delegations are stored using SQL backend, so using critical sections in any
request involving a call to delegation manager or driver should be carefully
tested to prevent deadlock.

Other Deployer Impact
---------------------

To use unified delegation drivers for assignment, trust or request token
instead of their own requires configuration change to *driver* parameter
in the corresponding configuration sections and apply data migrations
unifying existing delegation data.

Mirgration impact
-----------------

Data from assignment, trust and oauth backends is to be merged into a single
unified delegation backend, this will require specific migration scripts to
be developed and applied on upgrade.

Developer Impact
----------------

Developers should continue to use the existing APIs as designed.
Future work will involve exposing aspects of the delegations to the
end users of existing APIs.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Alexander Makarov amakarov@mirantis.com

Other contributors:
  Adam Young ayoung@redhat.com

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* This design is not currently dependant on other specifications.

* This blueprint and the implied roles blueprint are related.  Future
  work will build on the two specifications together.

* This specification does not require any additional libraries.

Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========


*  Implied Roles: https://review.openstack.org/#/c/125704/


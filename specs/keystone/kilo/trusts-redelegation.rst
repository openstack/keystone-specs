..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Allow Redelegation via Trusts
=============================

`bp trusts-redelegation
<https://blueprints.launchpad.net/keystone/+spec/trusts-redelegation>`_

Add support to trusts such that redelegation between consumers of trusts
(trustees, which are services for the currently expected use-cases) is
possible.

Problem Description
===================

Heat uses trusts to enable the service to act on behalf of a user when
performing deferred operations during the lifecycle of a Heat stack, for
example, when creating new instances in response to an autoscaling event.

The usage is as follows:

1. User (trustor) creates a Heat stack.

2. Heat creates a trust between the trustor and the Heat service user
   (trustee).

3. At some point after the stack is completed, Heat consumes the trust to
   impersonate the trustor, typically to perform some deferred task.

This poses a problem if the request creating the Heat stack comes from another
service which is also using trusts, as the token in the request will be scoped
to a trust, thus will be denied from creating the trust in step (2) above.

The main use-case for this is Solum, which wishes to use trusts in a similar
way to Heat, to perform deferred operations via trusts, which may include
creating Heat stacks, thus requiring creation of trusts enabling redelegation
from a trust-scoped token.

Another potential use-case is when Heat creates an alarm with Ceilometer, we
would like Ceilometer to create a trust, so it can impersonate the user owning
the stack when sending asynchronous alarm signals to Heat (which could trigger
creation of stacks, thus creation of trusts).

Proposed Change
===============

The proposal is that Keystone trusts will add optional support for chained
delegation, which would enable a trust to specify that redelegation is
permissible via a flag set on creation.

On creation of a trust, a new ``allow_redelegation`` argument may be specified,
which if omitted will default to False, meaning the current default behavior
remains where no redelegation is allowed.

If ``allow_redelegation`` is ``True``, then explicit redelegation of some or
all of the roles delegated by the trust will be possible, up to a configurable
maximum number of redelegations, tracked by a redelegation_count associated
with each trust, which is incremented each time a redelegation happens.

Thus, the following becomes possible:

1. Solum creates `trust1` with ``allow_redelegation=True``,
   ``redelegation_count`` is ``0``.

2. Solum creates a Heat stack with a trust-scoped token by consuming `trust1`

3. Heat creates `trust2` using the trust scoped token,
   ``allow_redelegation=True``, ``redelegation_count`` is ``1``.

4. Heat creates a Ceilometer alarm with a trust-scoped token consuming
   `trust2`.

5. Ceilometer creates a trust using the trust scoped token,
   ``redelegation_count`` becomes ``2``.

Proposed Change
===============

* Add a ``allow_redelegation`` interface to the OS-TRUST API specification.

* Update the trusts implementation adding ``allow_redelegation`` and
  ``redelegation_count`` to the DB and internal data model.

* Implement controller logic which handles the checking of
  ``allow_redelegation`` / ``redelegation_count`` when creating a trust.

* Implement chaining such that trusts created via redelegation are invalidated
  or when the parent trust is deleted or expires.

* When a trust is invalidated, all redelegated trusts are also invalidated.

Alternatives
------------

* Passing OAuth access/secret keys between services was considered, but have
  been discounted for now because there's no non-global way to disable
  accesskey expiry.

* Move to a model where one trust can have multiple trustee users (e.g more
  than one service can consume the same trust).

Data Model Impact
-----------------

One new DB column and item in the data structure representing a trust, similar
to "remaining_uses" which was added during Icehouse.

REST API Impact
---------------

One new optional item in the dictionary representing the trust,
``allow_redelegation``.

Backwards compatibility will be maintained by simply defaulting it to zero
inside the controller if it's not passed by the user.

Security Impact
---------------

None, other than allowing for explicit redelegation where none is currently
allowed.

To limit the maximum number of redelegations, there will be a new global config
option ``max_redelegation_count``, defaulted to a small number (such as 3) has
been suggested.

If ``redelegation_count`` would be greater than ``max_redelegation_count`` when
creating the trust, the request will be rejected.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Currently this is expected to be primarily useful for layered services
consuming trusts, but other use-cases could include supporting redelegation
workflow between human users.

There's no actual impact on the end user though, as the default behavior is
unchanged.

Performance Impact
------------------

The performance penalty for the additional logic to decrement the counter on
trust create should be negligible.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None, other than allowing full resolution of bug `bug #1317293
<https://bugs.launchpad.net/heat/+bug/1317293>`_

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  shardy (Steven Hardy)

Work Items
----------

* Update data model to add ``allow_redelegation``
* Implement controller logic for defaulting/decrementing ``allow_redelegation``
* Implement controller logic to cascade on expiry/delete
* Update ``python-keystoneclient`` to support ``allow_redelegation``


Dependencies
============

None, but related to fixing Solum/Heat `bug #1317293
<https://bugs.launchpad.net/heat/+bug/1317293>`_

Testing
=======

Writing a new tempest test will be a good idea. I'll do that as part of the
implementation, exact items to be tested TBC, but will probably include:

* Test default (no redelegation)
* Test a positive ``allow_redelegation`` and prove redelegation is prevented
  when it reaches zero via chained creation of trusts.
* Prove scope (project and roles) accessible after redelegation is the same as
  in the parent trust, and that no escalation of privileges is possible.
* Prove cascading invalidation on delete works
* Prove cascading invalidation works on expiry

Documentation Impact
====================

identity-api documentation will require an update.

References
==========

* http://lists.openstack.org/pipermail/openstack-dev/2014-June/036490.html
* https://bugs.launchpad.net/solum/+bug/1317293
* https://etherpad.openstack.org/p/icehouse-delegation
* http://dolphm.com/openstack-icehouse-design-summit-outcomes-for-keystone/

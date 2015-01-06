..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Scope federation tokens with ``token`` authentication method
============================================================

`bp federation-token-scoping <https://blueprints.launchpad.net/keystone/+spec/federation-token-scoping>`_

For a more consistent scoping procedure, unscoped *federated* tokens should be
carried out with the standard ``token`` authentication method.

Problem Description
===================

Currently unscoped federated tokens must be scoped with use of dedicated
authentication method - ``saml2`` (or more generic - ``mapped``). This can be
unified with a classic scoping workflow, hence authentication method ``token``
should also be used.

Proposed Change
===============

The proposed solution is two fold:

* Update documentation and indicate that preferred authentication method used
  for scoping federated tokens is ``token``, even though ``mapped`` and
  ``saml2`` will remain available.
* Properly handle scoping federated  tokens via ``token`` authentication
  method.


Alternatives
------------

Keep using existing authentication methods.

Security Impact
---------------

* Does this change touch sensitive data such as tokens, keys, or user data?

  Yes, but the change doesn't alter the already established behaviour that is
  present in OpenStack.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

  No.

* Does this change involve cryptography or hashing?

  No.

* Does this change require the use of sudo or any elevated privileges?

  No.

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

  Yes, but the change doesn't alter the already established behaviour that is
  present in OpenStack.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  No.


Notifications Impact
--------------------

None.

Other End User Impact
---------------------

All the clients will need to be changed so the new authentication method is
utilized. This also includes relevant changes in ``python-keystoneclient``.

Performance Impact
------------------

No performance changes.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------


Primary assignee:
    Marek Denis <marek-denis>


Work Items
----------

* Implement required change in Keystone
* Update documentation

Dependencies
============

None

Documentation Impact
====================

Documentation must be updated.

References
==========

Keystone change proposal: https://review.openstack.org/#/c/130593

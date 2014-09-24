..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Rescoping Spec - From Unscoped to Scoped
========================================

`bp rescoping <https://blueprints.launchpad.net/keystone/+spec/rescoping>`_



Limit the rescoping of tokens to prevent unauthorized reuse or escalation of
privileges.


Problem Description
===================

An unscoped token can be used to obtain a token scoped to any project or
domain. However, once a user is in possession of a scoped token, that user, or
any user, which obtains the token is able to exchange it for another project
scoped token. This should not be allowed, since it means that any token is
effectively unconstrained. In order to exchange a scoped token to an unscoped
token the user should have to re-authenticate to prove entitlement to the
unscoped token. A user should never be able to exchange one scoped token for a
differently scoped token.


Proposed Change
===============

Constrain the token creation process so that:
* A user must authenticate in order to obtain an unscoped token.
* Only tokens that are unscoped can be exchanged for a token that is scoped
to a domain or project.
* A token that is scoped to a domain or project cannot be exchanged for
another token.



Alternatives
------------

None:

Security Impact
---------------

* Does this change touch sensitive data such as tokens, keys, or user data?
* Yes:  token for token echanges are to be limited.


* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
* No


* Does this change involve cryptography or hashing?
* No

* Does this change require the use of sudo or any elevated privileges?
* No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.
* No


* Can this change enable a resource exhaustion attack.
* No


Notifications Impact
--------------------

None

Other End User Impact
---------------------

This should be hidden from the end user in just about every case.  The one
exception is people making direct CURL calls to Keystone.  The Keystone client
will make this seemless elsewhere.

Performance Impact
------------------

Many requests for a single, scoped, token will now require multiple tokens.
First one will be unscoped, second will be scoped.  Horizon will be the primary
consumer that sees this.  However, in LDAP deployments, Horizon already works
this way.


Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

A Config flag to enable and disable this feature

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

Needs to be explicitly enabled


* If this change is a new binary, how would it be deployed?

No


Developer Impact
----------------

All services that need multiple tokens will have to hold on to one unscoped
token to get the other tokens.  This should be managed by the keystone client.


Implementation
==============

Assignee(s)
-----------


Primary assignee:

ayoung  : Adam Young <ayoung@redhat.com>

Other contributors:


Work Items
----------

* Server side changes to deny rescoping a scoped token.  Initially disabled.

* Changes to Keystone client to maintain both the unscoped and scoped tokens.

* Changes to Django-OpenStack Auth to only attempt to rescope from an unscoped
  token.


Dependencies
============

*  specs/kilo/explicit-unscoped.rst

References
==========

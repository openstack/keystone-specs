..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Unscoped Token Catalogs
=======================

`unscoped-catalog <https://blueprints.launchpad.net/keystone/+spec/unscoped-catalog>`_

The flow of an unscoped token is to use it to communicate with the identity
server. Ensure that we can easily discover the appropriate identity service
endpoint when we start with an unscoped token.

Problem Description
===================

An unscoped token does not contain a service catalog. This means that even
though it is designed to be used to talk to an identity server it doesn't
follow the standard flow of querying the token for the URL or for the projects
or domains that are associated with the user.

This is really a pain from a client usability perspective because it means that
for certain requests if no catalog is available we should issue it against the
auth_url.

Proposed Change
===============

I propose we add a service catalog consisting purely of the identity endpoints
to unscoped tokens. The standard workflow by which you retrieve an unscoped
token, then list available projects (or domains) that can be scoped to then
retrieve a scoped token still applies, however now the URL that we use to
contact the identity service to list projects should be retrieved from the
service catalog rather than the original authentication URL.

Alternatives
------------

We continue to keep unscoped tokens without a catalog. There will be some
requests that may need to be issued differently depending on the presence of a
service catalog.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

There is a possible concern as in that users may be using the presence of a
service catalog within a token to tell if a token is scoped or not. This can be
considered bad practice and scope should be correctly determined.

Performance Impact
------------------

None.

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
    jamielennox

Other contributors:

Work Items
----------

* Add a service catalog to unscoped tokens. Clients would benefit from this
  immediately with our existing code.

Dependencies
============

None.

Documentation Impact
====================

We need to make it clear in documentation that this is a change from the
existing data and that people should always consult the service catalog.

References
==========

None.

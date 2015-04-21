..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Decouple Auth from API Version
==============================

`bp move-auth-url
<https://blueprints.launchpad.net/keystone/+spec/move-auth-url>`_


Currently Authentication is directly tied to the API version of Keystone. This
means that we use ``/v3/auth`` and ``/v2.0/tokens`` as the respective locations
for authentication. Authentication is a separate concern from the interaction
with Keystone's CRUD interfaces. To this extent, the location that handles
authentication should be separated from the API versions.


Problem Description
===================

With the move to isolate authentication (via the keystoneauth library) and
generally cleanup how Keystone works from a end-user perspective, it would
be an improvement to move to a ``/auth`` location to handle all authentication
requests.

This has the added benefit of decoupling authentication responses from a
specific API version. If an auth change is desired, the contract in data
returned is not locked in as it is today, a new auth version can be created
without changing the Keystone CRUD API, since auth is now decoupled.

Proposed Change
===============

The change proposed here is to add ``/auth`` (at the root) as the primary
location to handle authentication requests. Under this new location,
authentication will occur the same as it works today with the following
changes:

* Requested version of auth (e.g. v2.0, v3, etc) can be requested in the post
  data body itself.

* A get on ``/auth`` will display (for discovery purposes) what versions of
  auth are available.

* ``/auth/catalog`` will be used for getting the catalog for a given token
  with appropriate filtering.

* The default auth version will be v3, if no version is requested. The
  new ``keystoneauth`` library will be updated to always request the highest
  version it supports.

* Validation will require an auth version to know what the token data should
  look like. This would default to the default version a token would be
  issued with.

* Federation authentication would likewise move under ``/auth`` rather than
  having requirements to utilize resources under ``OS-FEDERATION``.

The current auth locations would be wired up on the backend to the new auth
system to ensure there is only 1 path authentication travels through. The
old locations for authentication will be deprecated (but no plans for removal)
until the whole API is deprecated (e.g. when V2.0 is removed, the
``/v2.0/tokens`` location will be removed, but not before).

.. note::

    The details on the specific API changes and more in-depth design will need
    to be addressed when this is moved from the backlog to the accepted spec
    location for a release.


Alternatives
------------

No direct alternatives. New auth mechanisms/changes to data returned or how
auth is handled would require a new URL location to handle requests in either
case.

Security Impact
---------------

This change impacts where Authentication happens and could lead to changes in
how tokens are handled. This means the new APIs will need to be evaluated for
security gaps. Generally speaking security should not be impacted or isolated
to a given auth version.

Notifications Impact
--------------------

Notifcations would not be impacted.

Other End User Impact
---------------------

* This will change how authentication works, however the old systems will
  continue to function. With the use of ``keystoneauth`` as a library, the
  change should end up seamless to consumers of the new library.

Performance Impact
------------------

No performance impact.

Other Deployer Impact
---------------------

Deployer specific impact should be minimal. External applications and services
not using ``keystonemiddleware`` (which will use ``keystoneauth`` library, and
therefore should be able to use the new auth mechanisms without extra overhead)
will need to understand the new auth locations and how to authenticate.

In most cases this should be minimal as the old authentication locations will
not be removed.

Developer Impact
----------------

Developers will need to be aware of how the new authentication APIs work.
However, the impact should be minimal as the old authentication locations
will continue to work. It will be recommended all auth start going through
the new APIs.

Implementation
==============

Assignee(s)
-----------

TBD

Work Items
----------

* Implement new ``/auth`` routes (API TBD when moved off backlog)

* Wire up old authentication routes to the new mechanisms so auth is handled
  in one place

* Update clients and middleware to use new ``keystoneauth`` library.

* Ensure ``keystoneauth`` library is capable of using the new auth locations.


Dependencies
============

This depends on ``keystoneauth`` becoming a real library and utilized in the
clients and middleware.


Documentation Impact
====================

Documentation on the new APIs will be needed.


References
==========



..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Retrieve Authentication Scoped Data
===================================

`bp auth-specific-data <https://blueprints.launchpad.net/keystone/+spec/auth-specific-data>`_

We need routes that allow us to retrieve information relevant to the current
authentication scope.

Problem Description
===================

Determining data associated with the current authorization scope requires
extracting information from the token and then constructing URLs based on that
information. This is a problem with calls for which the information is based on
multiple factors. It is also a problem for the client where we are actively
trying to encapsulate the authentication information so that you don't have to
know the parameters of your current authentication.

There has recently `been a change`_ to expose the service catalog based on the
current authentication token. However this is not the only thing that needs to
be determined on a per token basis. For example federated tokens need a
separate URL to determine the projects as there is no determinable ``user_id``
that can be put into the standard URL.

.. _been a change: https://blueprints.launchpad.net/keystone/+spec/get-catalog

Proposed Change
===============

I propose we repurpose the rest of the ``/auth`` URL to provide resources based
on the current authentication scope. This has a number of immediate URL
endpoints.

* Create ``GET /v3/auth/projects``. This lists the projects that a new scoped
  token can be requested for using the current token as authentication.  This
  route can be used as a replacement for both the federation and regular user
  based project lookup. It will allow for projects to be associated with the
  current authentication via more that just user association.
  Deprecate ``GET /v3/OS-FEDERATION/projects`` in favour of this.

* Create ``GET /v3/auth/domains``. List the domains that a new scoped token can
  be requested for using the current token as authentication. As with projects
  this combines the federated and regular domain lookup cases.
  Deprecate ``GET /v3/OS-FEDERATION/domains`` in favour of this.

I feel this also simplifies the security model because it means all requests to
``/auth`` are purely based on the current authentication scope rather than
dealing with matching filter arguments to the current scope.

It also opens up future options for matching available projects and domains
based on more than the ``user_id``.

Currently I am not proposing to add ``POST`` or modifying routes to the URL,
though these may be wanted later.

Alternatives
------------

Essentially we can continue down the current path. Add a new ``GET v3/catalog``
route that is useful but inconsistent with the rest of the API, and requires
the client to remember whether you need to list projects via either the
federation endpoint or the regular project list based on the token type.

Security Impact
---------------

None. All the information is currently available via existing APIs just in less
convenient ways.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

It simplifies things from a client perspective because you don't need to know
what sort of token you have to list projects/domains from. In general it
simplifies receiving information regarding the current token, both the existing
and anything we choose to expose in future.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

This will hopefully make redundant the ``/v3/OS-FEDERATION/projects`` and
``/v3/OS-FEDERATION/domains`` endpoints. We may want to redirect requests from
there to the new endpoints.

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

* Create new routes for projects and domains, join federation and listing APIs.
* Deprecate the corresponding OS-FEDERATION routes.
* Expose this information via keystoneclient.


Dependencies
============

* https://blueprints.launchpad.net/keystone/+spec/get-catalog

Documentation Impact
====================

New APIs. Possibly some changes to the documentation regarding the standard
workflow of scoping tokens.

References
==========

None.

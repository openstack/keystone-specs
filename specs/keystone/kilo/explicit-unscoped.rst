..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Explicitly Unscoped Tokens
==========================

`bp explicit-unscoped <https://blueprints.launchpad.net/keystone/+spec/explicit-unscoped>`_

The V3 API provides no way to request an unscoped token if the user has a
``default_project_id`` set.

Problem Description
===================

Keystone has the concept of a ``default_project_id`` which means that if a
scope is not specified on an authentication request the scope is assumed to be
the ``default_project_id``. This makes it impossible for a user with a
``default_project_id`` set to gain an unscoped token.

Proposed Change
===============

Add a mechanism to specify that an unscoped token is requested regardless of
the presence of a ``default_project_id``.

Alternatives
------------

None.

Security Impact
---------------

Ideally for security purposes we would like to maintain our explicit separation
of scoped and unscoped tokens. This could be the first step in removing the
``default_project_id`` and enforcing the unscoped workflow.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Keystoneclient will need to have a new option on its authentication plugins to
signal that an unscoped token is required. This should not change the typical
flow of an application as it would already need to handle the case where the
token it received was unscoped.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

This will lead to greater consistency in authentication flows.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    jamielennox

Other contributors:
    ayoung

Work Items
----------

* Add the change to authentication routes.
* Add the option to auth plugins in keystoneclient.

Dependencies
============

None.

Documentation Impact
====================

There is a new option available in the identity-api and a corresponding new
option on the auth plugins in keystoneclient.

References
==========

None.

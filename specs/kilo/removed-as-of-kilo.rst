..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
Deprecated items that are removed as of the Kilo release
========================================================

`bp removed-as-of-kilo <https://blueprints.launchpad.net/keystone/+spec/
removed-as-of-kilo>`_

Problem Description
===================

This spec is a list of deprecated items that will be removed in the Kilo
release.

Proposed Change
===============

See work items.


Alternatives
------------

None

Data Model Impact
-----------------

N/A

REST API Impact
---------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

Deployers will no longer be able to enable to the now removed functionality.

Developer Impact
----------------

Developers will no longer have access to the removed code.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    keystone-drivers

Work Items
----------

- Remove non-token KVS Backends and KVS-specific Testing (Identity,
  Assignment, Trust, and Revoke).
- Remove OS-STATS extension
- Remove XMLMiddleware
- Remove Token Persistence proxy classes
- Remove `token_api` manager class.
- Migrate deprecated `[signing]` options to oslo.config `DeprecatedOpt`
- Remove support for Auth Plugin Configuration by class name.
- Remove deprecated TemplatedCatalog class

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Any documentation referencing removed items will need to be updated.

References
==========

None

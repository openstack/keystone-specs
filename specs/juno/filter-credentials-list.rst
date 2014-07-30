..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Filter credentials by user ID
=============================

`bp example <https://blueprints.launchpad.net/keystone/+spec/filter-credentials-by-user>`_


Enable control of credentials for users via the Identity policy file


Problem Description
===================

Credentials can be created for users and stored in the Identity service, tagged
with the appropriate ``user_id``. The Identity service policy engine will
already allow a user to have control over their own credentials, for update,
get and delete operations.  However, in order to get a credential you must know
its ID, and the only way to get an ID is to list those credentials tagged with
your ``user_id``.  The only way to control such listing via policy is via
using the policy control of filters - but the list credentials API does not
support a filter on ``user_id``.

Proposed Change
===============

Add a filter of ``user_id`` to the list credentials API.

Alternatives
------------

None

Data Model Impact
-----------------

None

REST API Impact
---------------

The exact API specification will be defined as part of a review of an
changes to the Identity API, but will simply consist of adding a standard
filter options of user_id.

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

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
    henry-nash

Additional assignee:
    Alexey Miroshkin

Work Items
----------

- Update API specifications
- Implement the filter in controller code

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Update to the Identity API to list the new filter attribute.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Fix tests to run on non-SQLite databases
========================================

`bp tests-on-rdbmses <https://blueprints.launchpad.net/keystone/+spec/tests-on-rdbmses>`_


The Keystone tests need to be modified to work on non-SQLite database.


Problem Description
===================

Keystone tests currently will only run on SQLite. There are many reasons to
run the tests on other database::

* by default SQLite doesn't enforce foreign keys
* most deployments will use another database like MySQL or PostgreSQL
* every few months somebody asks how to make it work


Proposed Change
===============

1. Fix the tests to respect foreign keys. We currently have tests that can't
   work in a database where foreign keys are enforced.
2. Fix any other bugs that pop up when running on another database. This may be
   very big or very small. It's hard to tell without actually starting.
3. Make SQLite enforce foreign keys to prevent bugs from slipping by
   developers that don't run tests against a non-SQLite database.

Alternatives
------------

There are really no alternatives as this primarily just fixes the broken tests.
At the patch level there may be alternative implementations possible, but that
is out of the scope of this proposal.

Security Impact
---------------

None. This is just about changing the way that tests are written and executed.

Notifications Impact
--------------------

None. This is just about changing the way that tests are written and executed.

Other End User Impact
---------------------

None. This is just about changing the way that tests are written and executed.

Performance Impact
------------------

None. This is just about changing the way that tests are written and executed.

Other Deployer Impact
---------------------

None. This is just about changing the way that tests are written and executed.

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the driver API, discussion of how
  other backends would implement the feature is required.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek

Work Items
----------

TBD


Dependencies
============

None.


Documentation Impact
====================

None. This is just about changing the way that tests are written and executed.


References
==========

None.

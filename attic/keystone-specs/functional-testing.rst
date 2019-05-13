..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Functional Testing
==================

`bp functional-testing
<https://blueprints.launchpad.net/keystone/+spec/functional-testing>`_

Problem Description
===================

The new direction is to `move functional testing`_ out of tempest and into the
projects. This means that we need to provide guidance for how this is to be
done in Keystone. This spec answers the following questons:

1. What is considered functional tests and how are they different from unit
   tests?
2. Where should the tests be located?
3. When should these tests be run?
4. What tooling should be used to write the tests?

We've also decided at the `Kilo mid-cycle meeting`_ that the `Work Items`_
will be updated in subsequent commits to reflect new functional tests
scenarios that will be added. If someone decides that they would like to add
tests to verify feature X, they will submit a change to this spec to add a
subsection to `Work Items`_ based on the template.

This spec may lead to moving some unit tests into the functional tests, but is
not advocating any substantial changes to how unit tests work today.

.. _move functional testing:
    http://lists.openstack.org/pipermail/openstack-dev/2014-July/041057.html
.. _Kilo mid-cycle meeting:
    https://etherpad.openstack.org/p/keystone-kilo-hackathon

Proposed Change
===============

Definitions
-----------

shared scenario tests
 Tests that **must** pass on all Keystone configurations.
 (also known as 'shared tests')

configuration specific scenario tests
 Tests that require Keystone is configured in a specific way. For example, to
 run the federation tests Keystone must be configured to support federation.
 (also known as 'config specific tests')

Functional or Unit Testing
--------------------------

From the `move functional testing`_ post:

 Put the burden for a bunch of these tests back on the projects as "functional"
 tests. Basically a custom devstack environment that a project can create with
 a set of services that they minimally need to do their job. These functional
 tests will live in the project tree, not in Tempest, so can be atomically
 landed as part of the project normal development process.

What exactly is a functional test? If the answer of any of the following
questions is *'yes'*, then you have a functional test.

1. Does the test scenario need any specific components setup?
   (e.g. ldap, federation)
2. Does the test need to test a Keystone instance as a black box?
3. Does the test need to be run as a part of the test suite that runs against
   a variety of different configuations?

.. example directory structure:

Directory structures
--------------------

.. code::

   $ pwd
   /opt/stack/keystone
   keystone/tests/functional
   ├── federation
   │   ├── __init__.py
   │   └── {{test_something}}.py
   └── shared
       ├── __init__.py
       └── {{test_something}}.py

   2 directories, 4 files

Running Functional Tests
------------------------

Functional shared tests should be run by developers before submitting changes
to Gerrit. This should not be much of a barrier because those are tests that
need to run on any running Keystone configuration.

When developers are working on feature for a specific configuration they
should run the config specific tests.

To run all of the shared tests (unlikely that you want to do this):

.. code::

  tox -e functional

To run all of the federation config tests:

.. code::

  tox -e functional -- keystone.tests.functional.federation

How To Write Functional Tests
-----------------------------

The tests will be written in a style that is similar to unit tests. `tox`
will be used to run the tests. Test classes are subclasses to `testtools`.

The difference comes in when you start talking about the contents of the tests
themselves. Functional tests should be written using the black box model. The
test should have no knowledge of the backends being used. Ideally all tests
can be written using primarily `keystoneclient` and `requests` to interact
with Keystone.

Certain elements of the tests *must* be controllable via environment
variables. For example, the base URL for Keystone. This allows the developer
running the tests to point to any Keystone instance. Beyond the Keystone URLs
there is no official list of what must be configurable.

Alternatives
------------

I have not really investigated alternatives. This spec is intentionally a bit
vague so that we can innovate a little in this space.

Security Impact
---------------

None. This is about tests and does't directly impact production code.

Notifications Impact
--------------------

None. This is about tests and does't directly impact production code.

Other End User Impact
---------------------

None. This is about tests and does't directly impact production code.

Performance Impact
------------------

None. This is about tests and does't directly impact production code.

Other Deployer Impact
---------------------

None. This is about tests and does't directly impact production code.

Developer Impact
----------------

1. Developers will have to understand what functional tests are and where they
   go in the tree.
2. Developers will need to know how and when to run the tests.
3. Some existing unit tests may be moved to the functional test suite.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek

Work Items
----------

1. Update developer documentation.
2. Change the tox target for unit tests to exclude functional tests.

Testing Changes
---------------

Type of tests being added (e.g. federation)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A short description of the use cases being added.

Assignee(s):
<anyone>

Dependencies
============

Certain functional tests may require a specific environment to be available.
For example, to run the federation tests your Keystone instance will have to
be configured to use federation.

Documentation Impact
====================

The developer documentation will need to be updated to explain how and when to
run the functional tests. A new document will need to be created to explain
how to create and extend the functional tests.

References
==========

This is part of a bigger effort to rework the way Keystone that started in an
`etherpad <https://etherpad.openstack.org/p/keystone-test-restructuring>`_.

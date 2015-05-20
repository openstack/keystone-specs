..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Functional Testing Environments
===============================

`bp functional-testing-setup
<https://blueprints.launchpad.net/keystone/+spec/functional-testing-setup>`_


The new direction for functional testing is to `move`_ it out of tempest and
into the projects. This means that we need to provide direction and some sort
of framework for how this is to be done in Keystone.

This information was originally captured in `an etherpad
<https://etherpad.openstack.org/p/keystone-functional-tests>`_ while it was
being discussed.

.. _move: http://lists.openstack.org/pipermail/openstack-dev/2014-July/041188.html


Problem Description
===================

In `another spec <http://specs.openstack.org/openstack/keystone-specs/specs/
kilo/functional-testing.html>`_ we defined two types of functional tests.
Shared tests that can be run against any Keystone instance to verify its
behavior. Configuration specific tests that should only be run against a
Keystone instance that has been configured a certain way, e.g. federation.

These tests can be run against any Keystone instance by specifying the URL.
This means that a provider that has implemented federation with some domain
specific IdP can run the federation tests to see if there are any major
problems.

We also need a standard way to define expected configurations so that it is
easy for developers to standup test environments and easy for CI.

Not all created configurations will be in CI. We will have to pick based on
risk of bugs, popularity and CI resources. This spec doesn't talk too much
more about that.


Proposed Change
===============

Constraints
-----------

* stick to the methods/mechanisms used for adding functional tests in other
  OpenStack projects; this may be slightly different so that we can
  experiment with better solutions, but I don't want to be too far outside of
  the box
* it should be very easy to add new configurations that need to be tested
* the tests should be runnable by the gate for enforcement
* the tests should be runnable by developers using devstack
* the tests, when possible, should be runnable with a builtin server so no
  devstack is required (likely only the shared tests)

Concepts
--------

devstack vm (dsvm) configurations
  the scripts and configuration necessary to setup Keystone for testing a
  type of deployment (eventlet, Apache, federation, etc)

scenario tests
  the actual tests themselves; I am picturing this just using our current
  framework and tools used in unit testing (maybe with more focus on using
  the KeystoneClient API

shared tests
  the scenario tests that show the base behavior of the system; these will
  run under every configuration

Basic idea
----------

Configurations are actually implemented as devstack hooks. This allows them to
be easily used for gate testing. Each configuration will provide a script that
will restack a devstack instance. This restacking just automates the calls to
the gate hooks. This will allow developers to switch between configurations as
their testing needs change.

The tests themselves will be run as tox targets.

Possible configurations
-----------------------

These are things that we potentially want to gate against. There are many
possible configurations to test against. Below are just a few. Generally
speaking every run_tests.sh will run the standard tests as well as any tests
specific to that configuration.

* eventlet - deploy on eventlet
* Apache - deploy behind mod_wsgi
* federation - deploy on Apache using mod_shib

.. example directory structure:

Directory structures
--------------------

::

   dsvm
   └── {configuration name}
       ├── devstack
       │   ├── extras.d
       │   │   └── *.sh
       │   ├── files
       │   │   └── {...}
       │   ├── lib
       │   │   └── {...}
       │   └── local.conf
       ├── stack.sh
       ├── unstack.sh
       └── run_tests.sh

configuration name
  This is the directory that will hold all of the files necessary for
  a configuraion.

extras.d
  A directory of shell scripts that implement devstack plugins. This is not
  Keystone specific, but rather devstack specific. These shell scripts are
  really just calling to functions defined in 'lib' shell scripts instead of
  implementing significant logic.

files
  A directory containing files used by devstack. This is there things like
  configuration templates would go.

lib
  A directory containing supporting shell scripts that defined the logic
  used by plugin scripts.

local.conf
  This is the devstack configuration. If defines the plugins necessary for
  testing the configuration.

stack.sh
  Script used to initialize a devstack with a specific configuration. This will
  move any existing configuration (local.conf) out of the way before installing
  the one bundled with this dsvm configuration. We'll then call devstack's
  stack.sh.

unstack.sh
  Removes any dsvm configuration specifics configs and moves back anything
  that was moved by stack.sh. We'll then call devstack's unstack.sh.

run_tests.sh
  Script to call stack.sh, run the functional tests and call unstack.sh for a
  dsvm configuration.

Alternatives
------------

I have not really investigated alternatives. This proposal represents what I
learned from other projects and the changes I think are necessary to satisfy
the `constraints`_.

Security Impact
---------------

None. This is about test environments and doesn't directly impact production
code.

Notifications Impact
--------------------

None. This is about test environments and doesn't directly impact production
code.

Other End User Impact
---------------------

None. This is about test environments and doesn't directly impact production
code.

Performance Impact
------------------

None. This is about test environments and doesn't directly impact production
code.

Other Deployer Impact
---------------------

None. This is about test environments and doesn't directly impact production
code.

Developer Impact
----------------

Developers will have to learn and understand devstack/devstack-gate to some
extent if they wish to use the bundled configurations for functional testing.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek

Other contributors:
  <anyone>

Work Items
----------

1. create initial configuration for standard tests
2. create a configuration for federation tests
3. create experimental gate jobs for both configurations

Dependencies
============

A devstack instance is necessary to use the configuration scripts.

Documentation Impact
====================

The developer documentation will need to be updated to explain how to run
the scripts to setup the devstack configurations.

References
==========

The start of the implementation:

* https://review.openstack.org/#/c/151310/
* https://review.openstack.org/#/c/151311/
* https://review.openstack.org/#/c/139137/

Some of the references I used when writing the code for this spec:

OSC

* http://git.openstack.org/cgit/openstack-infra/project-config/tree/jenkins/jobs/osc-functional.yaml
* http://git.openstack.org/cgit/openstack/python-openstackclient/tree/post_test_hook.sh

devstack-gate

* https://github.com/openstack-infra/devstack-gate

neutron

* http://git.openstack.org/cgit/openstack-infra/project-config/tree/jenkins/jobs/neutron-functional.yaml
* http://git.openstack.org/cgit/openstack/neutron/tree/neutron/tests/contrib/gate_hook.sh

designate

* https://github.com/openstack/designate/blob/master/contrib/devstack/post_test_hook.sh

Google!:

* https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&es_th=1&ie=UTF-8#safe=active&q=devstack%20post_test_hook

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Devstack Plugin for Keystone
============================

`bp devstack-plugin
<https://blueprints.launchpad.net/keystone/+spec/devstack-plugin>`_

Problem Description
===================

Some functional tests require setting up a custom environment on which to run.

For example, federation tests require configuration of a service provider
module in Apache, and federation with an external identity provider such as
Shibboleth or another keystone (for keystone-to-keystone federation.)

Fortunately, Devstack includes a mechanism for allowing scripts to be plugged
in so they can be called by ``stack.sh`` during the different phases of
installation, making it ideal for preparing the custom environments we
require.[1]

The previous attempt[2] at writing a Devstack plugin for keystone was
overly confusing and difficult to test. To make reviewing and testing it
easier, it will be broken up into smaller pieces. These will be slowly rolled
out, incrementally adding the functionality required for the testing
environment.

The purpose of this spec is to have a reference describing the undertaken
approach.

Proposed Change
===============

This spec proposes adding an in-tree Devstack plugin for keystone. This plugin
will be enabled in the functional jobs.

The keystone plugin will be enabled by adding the following to the
``local.conf`` file:

.. code::

    # local.conf
    enable_plugin keystone https://git.openstack.org/openstack/keystone.git


If you are checking out for Gerrit, substitute the git url for
``$KEYSTONE_REPO``.

To make the plugin more configurable, the different features will be enabled
by adding ``enable_service keystone-<feature_name>`` to ``local.conf``. For
example, the following will enable federation features in keystone and make
keystone act as an identity provider:

.. code::

    # local.conf
    enable_service keystone-federation
    enable_service keystone-idp

Directory Structure
-------------------

.. code::

   /opt/stack/keystone/
    └── devstack/
        ├── settings
        ├── plugin.sh
        ├── files/
        └── lib/
            ├── federation.sh
            └── {{feature}}.sh


Plugins are expected to have a very specific structure. There should be a
top-level ``/devstack`` which minimally contains a ``plugin.sh`` file. This is
the file that Devstack calls into. [3] already added this minimal structure to
the keystone tree.

Additionally, there can be a ``settings`` file which is sourced very early to
provide defaults to the plugin. These defaults can be overwritten by
user-changes to ``local.conf``.

The keystone plugin will build upon this structure to include two more
directories.

``/devstack/files`` will contain preconfigured or templated files for external
services that need to be installed.

``/devstack/lib`` will contain scripts for specific features, organized into
files. For example ``/devstack/lib/federation.sh`` will contain the steps to
install and configure federation. The files in this directory will be sourced
and called from ``plugin.sh``.

Phases for federation
---------------------
[3] already added the minimal structure, and [4] will implement federation
with testshib.org working as the identity provider.

This is only a intermediary step, since we don't want to
rely on external identity providers for a critical piece of our testing
infrastructure. Federating with testshib.org allows us to build on something
that we know works, and is easily reproducible, allowing reviewers to more
easily and confidently review the plugin.

The next step would be to have a local instance on Shibboleth installed by
the plugin, be the identity provider. During the Ocata summit[5] there was
discussion to use a preconfigured docker image to simplify Shibboleth
deployment.

Once we're able to get the local Shibboleth to work with testshib.org
we can then plug it to keystone.

Finally, the next step would be to have keystone act as an identity provider,
and also support more than one identity provider.

Alternatives
------------

* Use orchestration tools to set up the testing environment.

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

Whenever a patch changes the way keystone is configured for a specific feature
which the Devstack plugin automates setting up, the plugin will need to be
updated.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  knikolla

Other contributors:
  rodrigods

Work Items
----------

* Add "hello world" Devstack plugin (done)
* Enable the plugin in the non-voting functional v3 job (done)
* Implement federation with testshib.org as the identity provider
* Implement federation with locally deployed Shibboleth as the identity
  provider
* Implement federation with multiple identity providers (might be a different
  job)
* Implement federation with keystone as the identity provider (might be a
  different job)

Dependencies
============

Documentation Impact
====================

References
==========

1. `Devstack plugin docs
<http://docs.openstack.org/developer/devstack/plugins.html>`_
2. `Previous Devstack plugin review
<https://review.openstack.org/#/c/320623/>`_
3. `Create structure for Devstack plugin
<https://review.openstack.org/#/c/395147/>`_
4. `Federate with testshib.org
<https://review.openstack.org/#/c/393932/>`_
5. `Etherpad for testing work session during the Ocata summit
<https://etherpad.openstack.org/p/ocata-keystone-testing>`_

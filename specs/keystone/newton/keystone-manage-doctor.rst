..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
keystone-manage doctor
======================

`bp keystone-manage-doctor <https://blueprints.launchpad.net/keystone/+spec/keystone-manage-doctor>`_

To provide a better deployer experience, let's provide a tool that deployers
can run to validate the state of a deployment and immediately receive
comprehensive feedback on suggested changes with detailed explanations.

Problem Description
===================

As upstream developers, operators, and summit attendees, many of us carry
tribal knowledge of various aspects of configuration that we recommend to other
deployers. We currently share that knowledge widely via documentation and blog
posts, which not all operators read. We also log deprecation warnings at
runtime.

- Are you using a deprecated configuration option?

- Is there an ``identity`` endpoint in your service catalog? Does it have a
  versioned endpoint? If so, which version?

- Does the ``is_admin_project`` you specified in ``keystone.conf`` actually
  exist in the backend?

- Does the ``default_domain_id`` you specified in ``keystone.conf`` actually
  exist in the backend?

Proposed Change
===============

Implement a new ``keystone-manage`` called ``doctor``. ``keystone-manage
doctor`` should diagnose issues with your deployment and make detailed
recommendations to resolve any issues.

Alternatives
------------

Keep preaching.

Security Impact
---------------

This command could be used to reveal security issues with a deployment.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

End users are not be able to use ``keystone-manage`` (only deployers).

Performance Impact
------------------

Running the command could place load on the keystone deployment, as it iterates
through large datasets looking for issues.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Similar to release notes, developers and code reviewers will need to be aware
of changes that should result in new checks that ``doctor`` should perform.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

- dolph

Work Items
----------

- Implement a no-op ``doctor`` command that doesn't actually perform any
  checks.

- Implement a set of checks for ``doctor`` to perform (perhaps some or all of
  those described in the Problem Description section above).

Dependencies
============

None.

Documentation Impact
====================

Operators should be made aware of the command's availability via release notes
and in openstack-manuals.

References
==========

None.

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Security Hardening: PCI-DSS
===========================

`bp pci-dss <https://blueprints.launchpad.net/keystone/+spec/pci-dss>`_

`Payment Card Industry - Data Security Standard (PCI-DSS) v3.1
<https://www.pcisecuritystandards.org/documents/PCI_DSS_v3-1.pdf>`_ provides an
industry standard for data security requirements and procedures. Although
keystone deals with sensitive data (primarily passwords), it has not made any
attempt to provide PCI-compliant tools to deployers for fear of re-implementing
more mature identity management solutions. At the same time, deployers are
taking on the additional burden of either deploying those fully featured
identity management solutions just to support keystone, or are re-implementing
these behaviors on top of keystone without community support.

Problem Description
===================

Keystone currently provides no means of satisfying *at least* the following
security hardening guidelines:

- **PCI-DSS 8.1.4**: Remove/disable inactive user accounts within 90 days.

- **PCI-DSS 8.1.6**: Limit repeated access attempts by locking out the user ID
  after not more than six attempts.

- **PCI-DSS 8.1.7**: Set the lockout duration to a minimum of 30 minutes or
  until an administrator enables the user ID.

- **PCI-DSS 8.2.3**: Passwords/phrases must meet the following: 1)
  Require a minimum length of at least seven characters. 2) Contain both
  numeric and alphabetic characters. Alternatively, the passwords/phrases must
  have complexity and strength at least equivalent to the parameters specified
  above.

- **PCI-DSS 8.2.4**: Change user passwords/passphrases at least once
  every 90 days.

- **PCI-DSS 8.2.5**: Do not allow an individual to submit a new password/phrase
  that is the same as any of the last four passwords/phrases he or she has
  used.

- **PCI-DSS 8.3**: Incorporate two-factor authentication for remote network
  access originating from outside the network by personnel (including users and
  administrators) and all third parties, (including vendor access for support
  or maintenance).

.. NOTE::

    There may be additional guidelines that Keystone does not currently
    satisfy.

Proposed Change
===============

`Shadow users
<https://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html>`_
moved user passwords into their own table, which coincidentally provides a
critical stepping stone to implementing all of the above behaviors.

To have the potential to achieve each standard "out of the box", we *at least*
need to make the following changes and associated behavior:

- **PCI-DSS 8.1.4**: Store the last successful login time per user name.
  Disallow authentication if that date is greater than a configurable time
  period (default: 90 days).

- **PCI-DSS 8.1.6**: Store rate limiting data of failed logins per user name.
  Deny logins if a configurable rate limit has been exceeded.

- **PCI-DSS 8.1.7**: We can permanently disable the identity using an existing
  disable attribute, or implement a rate limiting algorithm for **8.1.6** that
  achieves this behavior by design.

- **PCI-DSS 8.2.3**: Provide a configurable regular expression that passwords
  must match (using the ``pattern`` attribute in JSON Schema, for example).

- **PCI-DSS 8.2.4**: Store an expiration date for each password, and only
  authenticate users against passwords that have not expired.

- **PCI-DSS 8.2.5**: Store more than one password per identity. We could also
  provide a keystone-manage process to prune more than a configured maximum
  number of passwords per user, if desired.

- **PCI-DSS 8.3**: This will be implemented as part of another spec.

.. NOTE::

    There may be additional changes required to satisfy the specified
    guidelines.

Alternatives
------------

Deployers either do not depend on Keystone for these behaviors, and instead
back Keystone to an existing identity management solution, or implement these
behaviors on top of keystone in middleware.

This spec prevents deployers from having to deploy another identity solution
just for PCI compliance, and also prevents multiple operators from duplicating
each other's work any further.

Security Impact
---------------

This spec hardens password-based authentication according to PCI-DSS.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Keystone will be capable of presenting a new set of error messages for each new
behavior.

Exposing password expirations and last login timestamps is not critical for PCI
compliance, so there should be no API impact from this change.

Performance Impact
------------------

Authentication will require additional checks, although data from tables that
are already being read from, so performance impact should be negligible unless
the system is being abused.

Other Deployer Impact
---------------------

Several new configuration options will be added to Keystone to customize each
behavior. The default values could either reflect Keystone's current behaviors
(and we could simply document the recommended, hardened values), or reflect the
recommended PCI-compliant standards in the default values.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

None, yet.

Work Items
----------

Each PCI standard could be pursued independently, but it might be easier to
design them all at once, write a single schema migration to add the required
columns, and then write individual patches to implement each new behavior over
the course of an entire release cycle.

Dependencies
============

This spec directly depends on the backend refactoring provided by `shadow users
<https://specs.openstack.org/openstack/keystone-specs/specs/mitaka/shadow-users.html>`_.

Documentation Impact
====================

Documentation describing the parts of keystone deployers need to pay attention
to when ensuring PCI compliance would be invaluable.

References
==========

* `Payment Card Industry - Data Security Standard (PCI-DSS) v3.1
  <https://www.pcisecuritystandards.org/documents/PCI_DSS_v3-1.pdf>`_

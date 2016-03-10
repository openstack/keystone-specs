..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====
ldap3
=====

`bp ldap3 <https://blueprints.launchpad.net/keystone/+spec/ldap3>`_


In order to fully support python3, we need to provide an LDAP driver that also
supports python3 and all the features that keystone requires.
python-ldap+ldappool do not support python3.

Problem Description
===================

Keystone can't fully support python3 until we've got an LDAP driver that
supports python3. `python-ldap` doesn't support python3. Eventually python2
will go away or libraries that keystone relies on will stop supporting it.
Distros will stop including python2 by default.

Proposed Change
===============

Develop a new ldap3 driver using ldap3. ldap3 has python3 support and includes
connection pooling. Also, it's a more pythonic API and doesn't require a binary
shared library.

The new ldap3 driver will only implement read operations. We've deprecated
write operations in the existing ldap driver already so no need to implement
that in the new driver. There's only an "identity" ldap driver now and there
will only be an "identity" ldap3 driver.

The new ldap3 driver should support all the odd things that the old ldap driver
did so that deployers can switch to it. For example, the ldap3 driver should
support Microsoft Active Directory flags for the `enabled` attribute.

The original ldap driver will be deprecated in full.


Alternatives
------------

We could try to convert the existing ldap driver to ldap3. There are a few
problems with this: 1) The existing tests depend heavily on python-ldap
behavior so we'd have to deal with that while developing the driver. 2) Some
configuration options are based on python-ldap/ldappool parameters which may
not have a direct mapping to ldap3. 3) The existing driver supports both read
and write so there's extra and unnecessary work involved supporting write
operations.

We could try to get python-ldap/ldappool to support python3. I don't know how
much work this would be, and what's the point when ldap3 is better?


Security Impact
---------------

Developing a new driver is going to raise the possibility of security issues
just like we've had in the existing LDAP driver. The ldap3 driver needs to
support TLS and binding to the LDAP server for security. This change touches
sensitive data such as user data and passwords. We'll have to hash passwords
correctly just like the existing LDAP driver does.


Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

This new driver will need to enable connection pooling like the original driver
does.


Other Deployer Impact
---------------------

There will be a new identity driver that the deployer should switch to and
different configuration options.

The old `ldap` identity driver will be deprecated along with all the options.

Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  blk-u

Other contributors:
  None

Work Items
----------

* Implement ldap3 driver.


Dependencies
============

This introduces a dependency on the ldap3 package. It will be optional and only
required for deployers using the ldap3 driver. ldap3 is already in
global-requirements for some reason.


Documentation Impact
====================

New config options and driver.


References
==========

* ldap3: http://ldap3.readthedocs.org/

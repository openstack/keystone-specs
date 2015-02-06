..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Make storing of extra SQL attributes optional
=============================================

`bp extra-optional
<https://blueprints.launchpad.net/keystone/+spec/extra-optional>`_

Allow a cloud provider to disable the storing of "extra" SQL attributes.


Problem Description
===================

Our SQL drivers have for a long time supported clients piggybacking on our
formal entity attributes by just including extra attributes in the API call
(e.g. POST) - which we then store in the 'extras' column. While this is used
by a number of customers, it's a practice we don't want to encourage. Moreover,
allowing this could mean that cloud providers are unintentionally storing PII
information if the client includes such attributes in their API calls.

Proposed Change
===============

We will provide a configuration option to define whether our SQL drivers will
store extra attributes. For backward compatibility this will be set,
by default, to allow such storage. Disabling this option will support any
extra parameters to either be silently ignored (with a warning to the log) or
error in both read and write of SQL entities.

If this option is disabled on a system that already has extra attributes stored
this data will not be deleted - that is left as an out-of-band operation for
the cloud provider.

As an aside, although we do support extra attributes for LDAP drivers, the
storage of these is already optional by virtue of the fact that a mapping must
be defined in the LDAP section of the configuration file.

Alternatives
------------

None

Security Impact
---------------

This should close a potential security loophole.

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

    Henry Nash (henry-nash)

Work Items
----------

* Create configuration option
* Honor that option in the core SQL code which converts the extra attributes

Dependencies
============

None

Documentation Impact
====================

Update to the configuration.rst

References
==========

None

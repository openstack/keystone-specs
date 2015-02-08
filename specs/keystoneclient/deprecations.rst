..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============
Deprecations
============

`bp deprecations
<https://blueprints.launchpad.net/python-keystoneclient/+spec/deprecations>`_

keystoneclient needs a consistent way to notify applications that deprecated
behavior is being used.


Problem Description
===================

keystoneclient doesn't advertise deprecations consistently. All these
techniques are used:

* A comment in the code.
* The Docstring mentions it.
* Python's ``warnings`` module is used.
* A FIXME comment that says to deprecate it

When things aren't deprecated properly they're not really deprecated. A
comment in the code isn't going to be visible to developers. Developers can't
be expected to read the code to figure out what's deprecated.

Proposed Change
===============

keystoneclient will use a consistent way to advertise deprecations. The
proposed way is to use oslo's
`debtcollector <https://launchpad.net/debtcollector>`_. For example, when a deprecated function is going to be removed, use debtcollector.removals.remove.

::

    debtcollector.removals.remove(message="description")

The existing deprecations, comment deprecations, etc., will be changed to use
debtcollector.

Alternatives
------------

Use python's regular warnings module. debtcollector provides more consistent
messaging.

Security Impact
---------------

By deprecating things properly we'll be able to remove the code and have less
code to maintain with potential security problems.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

Depending on how the application has configured debtcollector already, they
may see the application fail, print errors, etc.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

Developers will use debtcollector for deprecations rather than warnings,
logs, etc.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <blk-u> Brant Knudson

Work Items
----------

* Change existing uses of DeprecationWarning to use debtcollector.
* Find places where comments or deprecations used to indicate deprecation and
  use debtcollector instead.
* Find places where FIXME was used to indicate to deprecate something and
  use debtcollector.


Dependencies
============

None.


Documentation Impact
====================

The keystoneclient documentation will be changed to say that the library uses
debtcollector for deprecations.


References
==========

* http://docs.openstack.org/developer/debtcollector/

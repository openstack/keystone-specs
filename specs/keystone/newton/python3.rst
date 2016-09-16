..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Python3.4 Compatibility
=======================

`bp python3 <https://blueprints.launchpad.net/keystone/+spec/python3>`_


Keystone should move towards Python 3.4 compatibility soon. Python 2.7 is
running towards it's end of life and we should make sure we are ready to move
to Python 3.4 and beyond as soon as possible.


Problem Description
===================

With python 2.7 being far away from the new development initiatives within the
Python community, Keystone (and more broadly) OpenStack needs to move to
Python 3 runtime compatibility.

Proposed Change
===============

There are a few python 3.4 incompatible libraries that need to be replaced with
libraries that are compatible with newer vintages of Python.

* Use the ldap3 library instead of python-ldap

* Use pymemcache instead of python-memcache

Any code that makes Python 2.x specific assumptions will need to be updated to
use be Python 3.x compatible.

Alternatives
------------

None

Security Impact
---------------

Python 3 in general is seeing new development, where python 2.x is not. This
means any security that is deep within the core of Python may be available
sooner.

There should be no direct security impact.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Performance profiles for Python 2 and 3 are different. In some cases the
performance will improve and in some cases it will deteriorate. It should
be a relative wash performance wise in the worst case scenario.

Other Deployer Impact
---------------------

Deployers will be able to use Python 3.4 instead of only Python 2.7

Developer Impact
----------------

Developers will be required to use only libraries and mechanisms that are
compatible with both Python 2.7 and Python 3.x. This is not wildly different
than today, just a little more work in some cases (such as handling
bytes types vs text types).


Implementation
==============

Assignee(s)
-----------

David Stanek <dstanek>
Morgan Fainberg <mdrnstm>

Work Items
----------

* Replace python-ldap with ldap3

* Replace python-memcache with pymemcache

* Identify and replace Python 3.x incompatible code in the keystone code base.


Dependencies
============

* Global requirements will need to include ldap3


Documentation Impact
====================

No significant documentation changes.

References
==========

* `pymemcache library <https://github.com/pinterest/pymemcache>`_

* `ldap3 library <https://ldap3.readthedocs.org>`_

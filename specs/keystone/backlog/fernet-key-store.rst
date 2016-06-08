..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Fernet Key Store
================

`bp fernet-key-store <https://blueprints.launchpad.net/keystone/+spec/fernet-key-store>`_

The existing Fernet implementation uses a file-backed key repository for
storing Fernet keys. A security optimization that can be made is to put the
keys into a dedicated key manager instead of having the Fernet keys on disk.


Problem Description
===================

Fernet currently doesn't support putting the keys used for encryption anywhere
except on disk. Providing a pluggable key manager would allow deployers to use
dedicated key storage tools to secure Fernet encryption keys.

Proposed Change
===============

There is already an existing interface defined as a `@property` object of the
`keystone.token.providers.fernet.token_formatters.TokenFormatter()` class. This
interface could be defined through a Fernet configuration option like
`CONF.fernet_tokens.backend`. By default the `backend` could be the existing
file-based implementation, but an operator could specify a different `backend`
using configuration. For example, Barbican or `Castellan
<http://docs.openstack.org/developer/castellan/>`_ could be used to store
Fernet keys.


Alternatives
------------

Continue to store keys on disk and use all the existing management tools.

Security Impact
---------------

Key rotation and distribution may change depending on the implementation being
used. This could be considered a security impact.

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

The key management tooling provided in ``keystone-manage`` may have to change
to support other key backends.

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <None>

Other contributors:
  <None>

Work Items
----------

Dependencies
============

Documentation Impact
====================

References
==========

`fernet crypto attribute <http://docs.openstack.org/developer/keystone/api/keystone.token.providers.fernet.html#keystone.token.providers.fernet.token_formatters.TokenFormatter.crypto>`_

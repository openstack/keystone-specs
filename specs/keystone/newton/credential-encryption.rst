..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Support Credential Encryption
=============================

`bp credential-encryption <https://blueprints.launchpad.net/keystone/+spec/credential-encryption>`_

Support encryption of credentials in Keystone to avoid having them stored in
plain text in the backend.

Problem Description
===================

Large organizations have security compliance that requires credentials to not
be stored in plain text.  Credentials in Keystone are currently being stored in
the backend and are accessible to anyone with access to the backend.
If a backend is compromised by an attacker they can easily get the credentials
for any user.  Also, anyone within an organization can look at the credentials
in the backend bypassing any security access controls offered by Keystone.

Proposed Change
===============

Update the credentials driver to support encryption of the ``blob`` field in
the ``credential`` table.  Given that there are viable ``secret`` providers out
there (Barbican, cryptography, etc) The choice of encryption solution should be
pluggable.  A single key will be used to encrypt the credentials.
Key management will be facilitated by allowing two active keys for decryption
and a single active key for encryption.  All credentials will be encrypted.
Key rotation will be done side band.  Keys are deployment wide.


Alternatives
------------

Barbican

Security Impact
---------------

Improved security of confidential information.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Encryption will decrease performance.

Other Deployer Impact
---------------------

Deployers will need to manage the keys.  Keys will initially be stored in
configuration files.  This will require a small amount of effort to setup.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  werner.mendizabal (Werner Mendizabal <nonameentername@gmail.com>)

Work Items
----------

* Update credentials driver to support encryption.
* Document upgrade process and how to enable encryption.
* Write keystone-manage command to encrypt existing credentials.  If deployer
  has not specified encryption keys, migration will not re-encrypt the
  credentials.
* Write tests to verify functionality.

Dependencies
============

* cryptography library for encryption.  Fernet will be the default encryption
  plugin.

Documentation Impact
====================

* Documentation should be updated to reflect configuration changes.

References
==========

* None

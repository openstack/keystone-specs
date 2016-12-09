..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================================
Auth plugin to support Multi-Factor Auth via password + TOTP
============================================================

related bp: `password-totp-plugin <https://blueprints.launchpad.net/keystone/+spec/password-totp-plugin>`_

related patch: `adding totp support to password auth plugin <https://review.openstack.org/#/c/343422>`_

Allow OpenStack to easily support optional Multi-Factor Auth (MFA) via TOTP
across the whole range of services.

Problem Description
===================

MFA is increasingly becoming a security standard, with TOTP being a good
choice for passcode generation. OpenStack via Keystone does not currently
support MFA and the current TOTP auth module is not useful for MFA.

A true MFA solution for OpenStack needs to be optionally enabled on a per
user basis, with enabled MFA either blocking the APIs or requiring the APIs
to accept an MFA passcode in some way.

Proposed Change
===============

Create an optional replacement for the default password auth method that
combines the logic of the current one, but will also do a TOTP check if
the user has a TOTP credential. This plugin will be optional and easily
enabled via a conf change, and should function entirely like the current
password auth plugin until a TOTP credential is added to a user.

Handle MFA by using the commonly accepted standard of concatenating
password+passcode. This allows MFA to work with the APIs and Horizon without
any work needed outside of Keystone.

This spec will only handle per user MFA. Per user MFA makes the most sense,
and it is how most people expect MFA to work. Attempting to handle per project
MFA would require work across all OpenStack services to be able to support it,
has a range of possible problems, and would require major refactoring in
almost all areas. As such per project MFA is not worth the effort until a very
good use case is put forward and until all other services are on board with
supporting it.

Dealing with MFA and V2 Token Auth
----------------------------------
If both V2 and V3 are active, then this change is useless as a good MFA
solution as someone could easily bypass MFA by authenticating against V2
instead. We will either need to update the V2 token auth to work with MFA
or simple deny MFA enabled users but work normally otherwise (editing mostly
deprecated code). The other option is to **very** clearly document that to
securely enable MFA, deployers **must disable the Keystone V2 Api**.
Suggesting to disable V2 is the easier option, with a possible follow up
spec to backport MFA to V2 if needed.


Alternatives
------------

Unknown

Security Impact
---------------

Improved security by allowing users with TOTP credentials to use MFA on their
accounts but depends on status of Keystone V2 api.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

No change to normal users, the change only causes different behavior once a
user has a TOTP credential associate with them, and even then the only change
is that to use Horizon or the APIs they need to append their TOTP passcode to
their password.

Unless we also update the V2 token auth, or the deployer has disabled it,
anyone using Keystone V2 will still be able to authenticate as per normal.

One impact is that users will no longer be able to use the standard openrc.sh
file as supplied by Horizon. Instead we should either replace the default in
Horizon with a multipurpose one that uses token auth, or add an optional one
specifically for MFA users.

Proposed replacement openrc.sh:
http://paste.openstack.org/show/565448/

Performance Impact
------------------

None

Other Deployer Impact
---------------------

Unless we also update the V2 token auth, deployers wanting to use MFA securely
will need to disable Keystone V2.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  adriant-y (Adrian Turjak <adriant@catalyst.net.nz>)

Work Items
----------

* Create password + TOTP auth plugin.
* Ensure the current username/password functionality works as expected.
* Write tests to ensure new functionality works as expected.
* Write documentation for activation and use of TOTP.

Dependencies
============

Not entirely dependent on, but a useful related change is:
- Credential Encryption

Documentation Impact
====================

Documentation needs to updated to show how users can activate and then use MFA
on their accounts while making it clear that MFA is optional and normal
non-MFA users are entirely unaffected.

References
==========

See email thread here:
http://lists.openstack.org/pipermail/openstack-dev/2016-July/099419.html

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
MFA Auth Receipt
================

`mfa-auth-receipt <https://blueprints.launchpad.net/keystone/+spec/
mfa-auth-receipt>`_


To provide proper usable multi-factor authentication (MFA) support in
OpenStack we need to return an auth receipt to the user representing partial
authentication, which can be returned to Keystone as part of a challenge
response process.


Problem Description
===================

As of Ocata Keystone supports setting auth rules on a given user and
potentially requiring them to auth with multiple methods. This is a good
implementation of MFA, but the problem rests in a user or service knowing ahead
of time how they need to auth. Most standard MFA systems are challenge response
based. You auth with one of your factors, then are given a receipt of that
partial authentication, which you can return along with the missing pieces.

Keystone needs such a mechanism because without it we can't realistically
consume MFA anywhere else in OpenStack in a manner that makes sense for users.

The original proposal for MFA in Keystone actually included this concept but
was never implemented:
`multi-factor-authn <https://blueprints.launchpad.net/keystone/+spec/
multi-factor-authn>`_

The concept of a half-token was discussed, but it never really got implemented
in a way we could use for MFA.

Proposed Change
===============

The proposal is to add a new mechanism which returns a receipt to the user on a
partially successful auth attempt. This would in essence be very similar to a
token, and ideally use a similar system for storage (fernet, jwt) to keep
internal logic consistent and reusable.

When a user successfully authenticates with one of their many auth rules, we
can infer that they are who they say they are, and then return to them a
receipt that contains information about missing auth rules.

If a user attempts auth with multiple methods and most fail but at least one
succeeds, we do not return a receipt, but we do include json error data as to
what methods failed, and which did not.

Using a receipt to get a new receipt with extra auth methods should ideally be
possible. If you need three auth methods to get a token, you should be able to
add to your receipt until it can become a token.

This receipt would allow no API interaction, but it would be proof that
certain auth methods have been completed. The expiry for the receipt would
default to a fairly short time-frame (5 minutes).

Receipt Response
----------------

The return code for this would be ``401`` as the user is still unauthorized,
with the response header containing the receipt id, and the body containing
information about the receipt which they need to continue the auth process.

In addition this response would be explicitly setup to only occur when a user
with auth rules successfully auths with one of the methods in any of their auth
rules.

The receipt id (the encrypted payload) would be included in the header of the
``401`` response in the same fashion as a token is but under the header
``OPENSTACK-AUTH-RECEIPT``.

An example response::

    {
        "receipt": {
            "methods": [
                "password"
            ],
            "user_id": "423f19a4ac1e4f48bbb4180756e6eb6c",
            "expires_at": "2015-11-06T15:32:17.893769Z"
        },
        "required_auth_methods": [
            ['password', 'totp'],
            ['password', 'option2'],
            ['password', 'option3', 'option4']
        ]
    }

``required_auth_methods`` would be a list of rules that include the
successfully authenticated methods and not a list of all of the user's auth
rules. The methods in the receipt do not entirely have to be all from the same
auth rule, the receipt contains valid methods, and what rules they could apply
to, so it's an intersection test.

Another example response::

    {
        "receipt": {
            "methods": [
                "password", "option3"
            ],
            "user_id": "423f19a4ac1e4f48bbb4180756e6eb6c",
            "expires_at": "2015-11-06T15:32:17.893769Z"
        },
        "required_auth_methods": [
            ['password', 'option2'],
            ['option3', 'option4']
        ]
    }

In this above case, the user now has two optional MFA paths off the same
receipt despite there being no overlap between the two auth rules.

Receipt payload
---------------

The receipt will not include scope data as it isn't relevant to it. A user will
supply scope when using the receipt and trying to get a token, Keystone will
not look at the receipt for scope. The only reason to potentially introduce
scope to the receipt is if and when we introduce auth rules based on scope.

The actual provider for the receipt should be configurable in the same fashion
as token, and similar types (fernet, jwt). In the case of fernet the key
repository should ideally be able to be shared with the token provider, but
configuration should be possible to split the repo. This minimises potential
pain for deployers and gives them time before they need to add a second key
repository.

Ideally any code we can share with the token providers we do, and move as much
as makes sense into common utils. Much like tokens, we even handle key
rotation in the same way. Accept receipts encrypted by the current and last key
but issue new ones with the current key.

The receipt data itself (before encryption) would likely be::

    {
        "methods": [
            "password"
        ],
        "user_id": "423f19a4ac1e4f48bbb4180756e6eb6c",
        "issued_at": "2015-11-06T14:32:17.893797Z",
    }

``methods`` in the above payload being the valid and already satisfied auth
methods for that receipt.

There isn't much data Keystone itself needs that it can't get from database
as long as it knows the user id. All we need to pass back is what methods have
already been validated, when it was issued at (for expiry checking), and the
user the receipt applies to (so we can match it with the user in the auth
request).

Receipt processing
------------------

When auth is continued with a receipt a user must supply the receipt id in the
``OPENSTACK-AUTH-RECEIPT`` header. When Keystone sees an auth attempt with
that header it will use the set receipt provider to decrypt the payload (id),
and will then treat the ``methods`` as defined in the receipt as having been
already satisfied as part of auth, and auth continues with the new user
supplied methods.

If all given methods are valid, but all the total valid methods including those
from the receipt still do not fulfill any of the auth rules, another receipt is
returned.

If the receipt is expired, don't even process the user supplied auth methods
and fail right away.

A mismatch between receipt user_id and user_id in any of the methods is a
failure.

A failure in any of the user supplied auth methods is a failure, and does not
extend the receipt even if some new methods were valid, but should include
failure details.

If a user supplies a method that was already satisfied in the receipt, the user
supplied method takes precedence, even if it fails.

All of these failures return a ``401``.

Alternatives
============

In Ocata when the auth rules were introduced the alternative was put forward
to simply change the error messages of the failed MFA auth to inform the user
why their auth failed. Any services could also parse the errors and infer
what the missing auth was and generate GUI or cli queries back to the user.

This doesn't work for a few reasons. While we can structure the response to be
parsable it still requires that for every auth attempt ALL methods are
supplied. This means an MFA attempt for password+totp needs to cache the
password to resubmit it when it is determined that totp is also needed.

For Horizon that doesn't work. We can't cache the password server-side, and we
shouldn't store it in a cookie or in browser really, and asking the user to
supply their password again isn't good UX.

This alternative also isn't a real challenge response.

Security Impact
===============

A user who authenticates with one successful method now knows what auth rules
can be used for a given user. This isn't particularly unsafe, as while they
know the rules, they can't auth with them. Additionally it is safe to assume
that one successful auth method is enough to prove identity of the user, while
not enough to satisfy authentication requirements.

If any method fails, we still return an error, and auth rules are only exposed
if at least one method was valid.

An additional security concern is that a given receipt could be used to make
multiple tokens in the short window it is valid. Given that users can already
have multiple tokens, this is only a concern if a receipt is intercepted and
completed by someone else. Given though that this receipt exists in the context
of MFA, the second factor should limit this concern quite heavily. In reality
this isn't a problem unique to the auth receipt, but an eventual action we
should take is to do some form of receipt revocations when turned into a token.

Notifications Impact
====================

Notifications will continue to be issued for successful and fail authentication
as today. Issuance of a receipt will not issue a notification.

We may add receipt notifications in the future.

Other End User Impact
=====================

The user may now process the returned data and utilize the receipt to
authenticate in multiple steps.

It is important to note that this change does not affect anyone only doing
auth with no auth rules set. The default auth use case does not change, only
the MFA use case which is not yet fully implemented or used anyway.

Performance Impact
==================

It is expected load and auth-processing-time will increase a small amount due
added encryption and decryption of the receipt.

Other Deployer Impact
=====================

Depending on how we handle fernet/jwt key repositories, deployers may need to
add a second key repository. Ideally we'd allow this to be configurable so at
first they didn't need to do this, but had the option to for added security.

Developer Impact
================

This would only affect authentication workflows in OpenStack and any code
dealing with them. This change though will give us a good programmatic way to
deal with MFA and will mean those tools can handle failures in a useful way.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * Adrian Turjak - adriant@catalyst.net.nz (IRC adriant)

Work Items
----------

#. Add support for the auth receipt to Keystone and change the auth plugin
   layer to return it on one successful auth method out of configured rules
   rather than an error, and make the auth layer able to process a receipt for
   continued auth.
#. Add support to KeystoneAuth to handle this correctly and infer from this
   receipt what other methods are needed, as well as adding proper Multi-Method
   support into KeystoneAuth.
#. Work with the CLI and Dashboard teams to build support in for MFA using the
   changes made to KeystoneAuth. (optional follow up)

Dependencies
============

N/A

Documentation Impact
====================

Will require extra documentation for new features, but no existing non-MFA auth
paths will change.

We need to add to the docs that MFA receipts also need encryption keys, and
that by default they are shared with fernet. Will need notes added that fernet
key rotation affect the receipts if their repository is shared.


References
==========

* https://adam.younglogic.com/2012/10/multifactor-auth-and-keystone/
* https://blueprints.launchpad.net/keystone/+spec/multi-factor-authn
* https://specs.openstack.org/openstack/keystone-specs/specs/keystone/ocata/per-user-auth-plugin-requirements.html

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Keystone Lightweight Tokens - KLWT
==================================

`bp klw-tokens <https://blueprints.launchpad.net/keystone/+spec/klw-tokens>`_

KLWT tokens provide a way to represent a token, allowing for tokens to be
non-persistent. KLWT tokens provide integrity and confidentiality, when
optionally using encryption, by being signed by Keystone. KLWT tokens contain
some amount of token information, including the user, the issued at time, the
expiration time, and the digest of the signed information. Regardless if
encryption is used, the tokens are authentic since they are signed and
validated by Keystone.


Problem Description
===================

After having deployed Keystone for some time, deployers have run into issues
with the amount of tokens stored in the token backend. To avoid this
problem, deployers must routinely clean out stale tokens from their
persistent storage to prevent performance or upgrade issues.

Proposed Change
===============

One way to avoid issues with massive amounts of tokens stored in a backend is
to not store tokens at all. KLWT tokens will provide a way for deployers to
avoid performance degradation and maintenance of tokens backends by using an
KLWT token driver. When authenticating, the KLWT token provider will pull a
subset of information about the user out of the request, generate a token
containing this data and return it to the user. To ensure we keep the tokens to
a reasonable length, the information can be compressed.

Although not to be considered prescribed by this spec, an example of this would
look similar to the following::

  payload = [scope_type, scope, user_id, created_at, expires_at, audit_id]

Where the values are:
* ``scope_type``: Scope type to determine the type of the token
(i.e. domain scoped token, trust scoped token, etc)
* ``scope``: Scope of the token, consisting of a trust ID, project ID, etc.
* ``user_id``: ID representing the user creating the token.
* ``created_at``: UTC timestamp representing the created at time of the token.
By using timestamp we can save on space when packing the message by packing an
integer instead of a string.
* ``expires_at``: UTC timestamp representing the expiration of the token. This
can be represented different ways and also using an integer to store the
timestamp instead of a string. This could optionally be a delta integer that is
the token ttl.
* ``audit_id``: The audit ID for the token.

Federation tokens will be a special case. We will need a specific formatter to
handle federated tokens. The difference between the KLWT provider/driver and a
KLWT token formatter is that a formatter knows how at handle a specific token
format, like federated tokens. The KLWT provider knows how to determine a token
format and pass the token to the proper KLWT formatter. We can determine if we
are authenticating in a federated workflow and use a format version specific to
federated tokens. In that specific format version, we can append the groups
necessary for federated tokens.

These values can be stored in a list where the KLWT Token driver, or any
validator for that matter, knows the order of the values. The above values can
be packed, possibly encrypted, and encoded. This will result in a ``payload``.
To ensure the integrity of the contents, the ``payload`` must be signed with a
mechanism like HMAC. The resulting digest will be included as part of the
token.

Alternatively, encryption can be used if the deployer wishes to keep
information opaque from end users. The important part is that the token data is
signed by Keystone for integrity regardless if the token is encrypted or not.
This is important because we should fast fail if the token has been tampered
with in any way without having to attempt decrypting the spoofed token only to
fail on bogus token values.

The user can then use a KLWT token in a normal UUID workflow. When Keystone
receives an KLWT token from a user or a service, it can validate the digest
provided with the token to determine if the token has been tampered with in
transit. This doesn't require the KLWT token driver to look up any information
from a token backend. The information traditionally stored in the token
backend, like the service catalog, will be computed on demand based on the
scope that is passed in the token.

This driver will only require minimum data to verify the user's identity. The
data will be included in the user's token. The workflow for the user doesn't
change from the current UUID workflow. The token size will be bigger than UUID
tokens but not as big as a PKI tokens.

That's neat, but how do I revoke an KLWT token that isn't stored?

Token revocation will be handled through revocation events, which is to be
built off the existing `revocation events
<http://docs.openstack.org/developer/keystone/extensions/revoke.html>`_ system.
When a user token has been revoked, the KLWT token driver will build a rule
that will describe a token to revoke. An example being, revoke all tokens from
user XYZ before this point in time. The next time the user sends the now
revoked token to Keystone, the validate call will check the token information
against the revocation rules and if a revocation rule matches the information
included in the token, return 401. Revocation events can be cleaned up anytime
after the default token expiration time has lapsed, since the revocation rule
was created. This work and framework is already in place, but the middleware
just needs a way to get the latest revocation events.


Alternatives
------------

None

Security Impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

  Yes, identity and authorization data will be optionally encrypted, compressed
  and returned to the API user instead of stored in a token backend.

* Does this change alter the API in a way that may impact security, such as a
  new way to access sensitive information or a new way to login?

  It shouldn't, all the same data will be required for authenticating and
  validating against Keystone.

* Does this change involve cryptography or hashing?

  Yes, this change involves cryptography to protect the user's information and
  hashing to verify. Keyczar can be used to handle KLWTS encryption.

* Does this change require the use of sudo or any elevated privileges?

  No

* Does this change involve using or parsing user-provided data? This could be
  directly at the API level or indirectly such as changes to a cache layer.

  The KLWT token will be built from data provided from the user. This doesn't
  make Keystone vulnerable to injection attacks since the information used will
  be pulled from a backing store.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  KLWT tokens do not make it possible to cause an exhaustion of resources from
  a persistence perspective, like UUID or PKI[z] tokens do becuause they don't
  have to be persisted.


Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

By using an KLWT token driver, Keystone performance should not decay as the
deployment size increases. In large Keystone deployments where tokens are
persisted to a backend, a performance hit may be observed as token creation
increases in scale. Instead of scaling a persistent storage solution for
tokens, an alternative would be to use KLWT tokens and scale up compute power.
Since the tokens never have to be written to a backend, CPU operations will
have better performance versus I/O operations.

Performance results have been `documented
<http://dolphm.com/benchmarking-openstack-keystone-token-formats>`_ based on a
`proof of concept KLWT token implementation
<https://review.openstack.org/#/c/145317/>`_.

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

  The KLWT token driver should be specified in the Keystone configuration file.

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

  This would be an opt-in feature and not enabled by default.

Developer Impact
----------------

* If the blueprint proposes a change to the driver API, discussion of how
  other backends would implement the feature is required.

  This change shouldn't be specific to a driver, from a persistent storage
  perspective.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lbragstad (Lance Bragstad <lbragstad@gmail.com>)

Work Items
----------

* Clean up the current token provider api such that it's easier to extend to
  different token formats.
* Allow revocation events for KLWT tokens (i.e. pattern matching)
* Add a new token driver for KLWT tokens and test.
* Document the process for setting up an KLWT token implementation.

Dependencies
============

* Support for revocation events that work with a KLWT token implementation.
* The base implementation is dependent on `python-keyczar
  <http://www.keyczar.org/>`_

Documentation Impact
====================

Detail how to switch to a KLWT token implementation and setting up the drivers.

References
==========

None

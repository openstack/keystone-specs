..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Token Provider Cleanup
======================

`token-provider-cleanup <https://blueprints.launchpad.net/keystone/+spec/
token-provider-cleanup>`_

Token providers should be a very simple class to create for anyone looking to
develop a custom provider. This opens the door(s) for allowing many token
providers both in-tree and out of tree while maintaining the token issuance
API contract.

Problem Description
===================

As of today to create a new token provider (with a new ID format), it requires
a lot of knowledge of how Keystone handles token issuance and what all the
raw data should look like. Keystone should have a very simple and easy to
understand interface for token providers. This allows new token-id formats to
be easily created that can/will meet specific deployer needs.

Proposed Change
===============

Keystone's token providers should be loaded with stevedore and have an
extemely stable API. This API should be very stable (enforced as a stable
contract in the same manner we enforce the REST API contract).

The new interface should be:

* **Issue Token**: A method that accepts a clear data model (similar to the
  current token data model introduced in Juno) and allows for extracting the
  data in a way that can be used to generate a Token ID. The most extreme case
  of this would be the PKI token where all of the data is encoded into the ID
  itself.

  The issue token method would also make decisions such as persisting the token
  data to some sort of stable storage and/or pre-seeding caches.

  A Token-ID type will need to be encoded into the Token ID (for new token
  types) allowing Keystone and Middleware to load the correct provider(s) to
  handle validating the token. Longer term this allows us to support multiple
  ID types (seamless conversion).

* **Validate Token**: A method that performs the validation and either utilizes
  Keystone's managers/etc to regenerate the token data or to extract the data
  directly (in the case of something akin to the PKI token model).

  Validate Token will be responsible for running the data through the token
  data formatter if needed.

* **Middleware Plugin**: A class that can be utilized by Keystone middleware to
  validate the token. A default behavior will be to utilize the UUID model
  where middleware performs an HTTP request to Keystone to validate the token
  and receive the token data / body back. Likely most providers will not re-
  implement this. An example provider that will re-implement this will be the
  PKI token provider.

  Middleware Plugin will use a method in common with `Validate Token` to avoid
  retrieving the token data from the persisted store more than once.

The V2 and V3 token helper classes will be refactored to take the token model
and emit the documented token data format. The token data formats will be
documented for developers and formatter classes for each version will be
provided (currently V2 and V3).

Any changes to this API will either need to be 100% compatible (no way to cause
an exception to be raised) or require an Adapter class to handle older provider
interfaces.

Alternatives
------------

None. This is a change of design, the alternative is to maintain the same
design we have today.

Security Impact
---------------

This change does affect tokens and potentially could have a security impact,
however, the goal is to maintain the same functionality but refactor the way
in which Keystone manages token issuance and token validation.

The new token provider(s) will need to be evaluated for security.


Notifications Impact
--------------------

There will be no notification impact. Notifications will be handled outside of
the provider `Issue Token` method ensuring the new providers do not need to be
aware of notification/cadf logic.

Other End User Impact
---------------------

Deployers will be able to develop their own token-id formats and validation.
This means token IDs may change what they look like. These changes should
be transparent to the end-user.

Performance Impact
------------------

Performance should have no direct impact from this change. This will, however,
allow deployers to select the best token provider for their deployments.

Other Deployer Impact
---------------------

Deployers will be able to choose the best token provider for their deployment.
Future development will allow for multiple providers (validation) so that
changing from one provider to another should be seamless to the end-user.

Developer Impact
----------------

Developers will have a very strictly contracted API to develop providers
against. This should improve developer experience significantly.


Implementation
==============

Assignee(s)
-----------

Morgan Fainberg <mdrnstm>

Work Items
----------

* Develop enhanced ABCMeta type class for enforcing the contract on token
  providers. This will include method signatures on required methods.

* Refactor token formatter classes that can convert token data to token model
  and vice-versa for V2 Token and V3 Token Formats. These formatters will live
  in Keystone and be made available as public interfaces for any token
  providers.

* Consolidate token issuance pipeline to not be version-specific.

* Develop framework for new token providers. This includes loading from
  stevedore.

* Implement token providers for the currently supported token formats (UUID,
  PKI, PKIZ)


Dependencies
============

No extra dependencies.


Documentation Impact
====================

* Documentation on the token provider interface will need to be developed.

* Token format documentation will need to be developed. Token data format will
  be marked as an internal "Keystone" data construct.

References
==========

No external references.

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Include invalid password details in audit messages
==================================================

`bug #2060972 <https://bugs.launchpad.net/keystone/+bug/2060972>`_

PCI DSS requires operators to analyze failed login attemps, for example, to
catch bruteforce or password stuffing attacks. To achieve that, allow Keystone
to report details about the invalid password used in the failed authentication
attempts.

Problem Description
===================

PCI DSS requires operators to analyze failed login attemps, for example, to
catch bruteforce or password stuffing attacks.

Keystone can be configured to report about failed authentication attempts, for
example, via ``identity.authenticate`` messages with outcome ``failure``, so to
further notify operators to perform the necessary analysis.

.. _problem-examples:

But experience shows, that some portion of repeated failed authentication
attempts happen not because there is a bad actor, but because user's automation
misbehaves, for example:

* A user updated their password in a backend, but have a script executed
  regularly (e.g. via cron) that is still using old password.

* A user automation has a bug that truncates a password, causing generation of
  failed authentication events on every run.

* A user ran a loop in a bash script but mistyped their password in the
  ``OS_PASSWORD`` variable for the openstack cli, making every loop run to
  generate a failed authentication event.

Notification messages produced by Keystone for such and similar cases still
need to be evaluated by operators in a same way as other messages, even though
they are not related to attacks.

With increasing number of such false positive events, operator's work is also
increasing, making it qualify as `toil
<https://sre.google/sre-book/eliminating-toil/>`_ and could lead to `alarm
fatigue <https://en.wikipedia.org/wiki/Alarm_fatigue>`_.

Proposed Change
===============

Enrich ``identity.authenticate`` events with outcome ``failure`` that resulted
from invalid password submission with a hash of the submitted (invalid)
password. The corresponding hashing function would be returning:

* **Same hash for same invalid passwords**. Indicating the cases described in
  :ref:`Problem Description examples <problem-examples>`. Example:

  .. code-block:: python

     > [generate_partial_password_hash(pwd, **kwargs) for pwd in ["invalidpwd0"] * 4]
     ['eyWwQ', 'eyWwQ', 'eyWwQ', 'eyWwQ']

* **Different hashes for changing password values**. Indicating e.g. bruteforce
  attacks. Example:

  .. code-block:: python

     > [generate_partial_password_hash(pwd, **kwargs) for pwd in ["invalidpwd0", "invalidpwd1", "invalidpwd2", "invalidpwd3"]]
     ['eyWwQ', '/qk6A', '9sU2g', 'OrVBw']

* **Partial hash value**. Making hash values of passwords more resistant to
  `Rainbow tables <https://en.wikipedia.org/wiki/Rainbow_table>`_, regardless
  that only *invalid* password hashes are produced.

Presence of such partial hash would allow to set up monitoring software to
group the events based on user and hash value, and set up limits for alerting.
For example, here is some *pseudo-code* to select all events of failed
authentication with varied hashes (and varied password correspondingly) over
1h:

.. code-block:: sql

   SELECT user,
          Count(DISTINCT partial_password_hash) AS distinct_hashes
   FROM   failed_auth_events
   WHERE  timestamp < now - 3600
   GROUP  BY user
   HAVING distinct_hashes > 1

In this example, the returned users would be the users that could have
experienced an attack. Conversly, the users that submitted the same invalid
password multiple times (as in :ref:`Problem Description examples
<problem-examples>`) won't be returned, as the corresponding hash won't be
changing. This would help operators to only focus on relevant events.

The inclusion of the hash would be  optional, disabled by default.

When enabled, payload of events of type ``identity.authenticate`` with outcome
``failure`` would be enriched with ``N`` characters of a hash of an invalid
password resulting to this event. See :ref:`notifications-impact` for details.

A configuration option would be available to configure ``N`` - number of
characters of a hash of an invalid password that would be returned. Additional
configuration options would be available to protect and secure hashes (e.g.
secret key [#owasp-cheat-sheet-peppering]_) produced by the hashing function.
See :ref:`other-deployer-impact`.

It's worth highlighting that only invalid passwords hashes would appear in
notification messages. ``identity.authenticate.success`` messages, for example,
won't include (valid) password hash.

Alternatives
------------

1. **Additional logging**. An option was considered to add additional log
   messages with details about authentication failures. Checking application
   logs is an accepted way in PCI DSS anomaly detection. However, keystone does
   not offer log stability and there are no libraries for proper log parsing.
   Audit messages are already a great way to log authentication failures, they
   already include some details about why authentication failed, and extending
   them seems to be a natural choice.

2. **A custom backend for password authentication**. One could create a custom
   password authentication backend. The backend would inherit from the vanilla
   keystone backend, catch authentication failures and log a message, emit a
   notification, or perform an action specific to the operator's needs.

   However, this will break other existing custom authentication plugins. It is
   also beneficial to have the details together with already emitted
   notifications, because there is already a way to process them.

3. **Implement the proposed change, but don't reveal hashes**. Instead of
   enriching messages with partial hash of a invalid password, Keystone could
   keep such hashes to itself (e.g. in some internal storage/memory), and
   provide configuration parameters setting up thresholds for a number of
   consequtive varied hashes, after which a failed authentication messages
   would eventually be enriched with additional informative text to notify
   operators about possible attack on a user, without exposing the hashes.

   The advantage would be that a partial hash of an invalid password is not
   exposed/stored in any form in a persistent storage, thus no attack could be
   performed against it.

   The disadvantage would be that distributed Keystone installations working
   with the same backend won't be able to aggregate the ``failed`` attempts'
   hashes across the whole "global" deployment, as each installation is
   independent and doesn't have shared internal storage that could be used.
   This fact could be then exploited by an attacker to avoid threshold
   violations, e.g. by directing each subsequent request to another
   installation, making each individual installation to not fire a
   notification, even though the global count of varying hashes would be above
   the healthy threshold.

.. _security-impact:

Security Impact
---------------

If an operator does not enable the feature in the config, there is no
security impact.

If the feature is enabled, it will expose ``N`` characters of hash of invalid
password. These characters will not be communicated with the response returned
to user, but will be sent via messages.

An operator with access to messages could perform an "attack" on the hash of
invalid password, which, if successful, would provide them with numerous
*invalid* password value possibilities that a user might have submitted to
produce that particular hash, that could potentially be further useful in a
preparation of a bruteforce attack against that user.

Administrators are to be advised to configure to return the least reasonable
number of characters ``N`` of a hash (see :ref:`other-deployer-impact`) - this
would be a very effective measure to protect hashes. With the default hash
algorithm in the proposed implementation (see :ref:`implementation`), a full
hash length would be ``43`` characters. ``N=5`` (to be provided by
administrators) could be an example of a value that's very secure, but still
resistant to collisions.

Note: event messages with not changing partial password hash do not guarantee
that there is no attack. It could mean that an attacker, using rainbow tables,
tries only passwords with hashes including the reported characters.

Operators must make their own choice how to interpret these hashes, and select
the number of the characters according to their needs and tooling.

.. _hash-algorithms-proposal:

In order to minimize performance impact, the proposal is to apply **HMAC** -
hash-based message authentication code [#hmac]_, instead of key derivation
functions [#kdf-password-hashing]_. The latter are more suitable for secure
password storage and validation. The former is also used by industry tools such
as HashiCorp Vault to hash sensitive data in audit logs
[#hashicorp-vault-audit-hashing]_.

.. _notifications-impact:

Notifications Impact
--------------------

When the feature is turned on, a new Attachment [#pycadf-attachments]_ would be
added to the notification events with event_type ``identity.authenticate`` and
outcome ``failure``. Event attachment name would be ``partial_password_hash``,
typeURI would be ``mime:text/plain``, with content representing partial hash of
a invalid password. The string would be 1 or more ascii characters. Example:

.. code-block:: yaml

   event_type: identity.authenticate
   payload:
     attachments:
     - content: z4Eya
       name: partial_password_hash
       typeURI: mime:text/plain
     outcome: failure
     ...

Other End User Impact
---------------------

None

.. _performance-impact:

Performance Impact
------------------

When the feature is turned on, hashing of a invalid password would be taking
minimal compute resources, depending on the configuration parameters used (see
:ref:`other-deployer-impact`).

.. code-block:: shell

   # default configuration
   python -m timeit -n 1000000 \
     -s "from keystone.common.password_hashing import generate_partial_password_hash" \
     "generate_partial_password_hash('invalidpwd0', 'strongsalt', secret_key='secret_key', max_chars=5, \
         hash_function='sha256')"
   1000000 loops, best of 5: 2.43 usec per loop

   # using HMAC with sha512
   python -m timeit -n 1000000 \
     -s "from keystone.common.password_hashing import generate_partial_password_hash" \
     "generate_partial_password_hash('invalidpwd0', 'strongsalt', secret_key='secret_key', max_chars=5, \
         hash_function='sha512')"
   1000000 loops, best of 5: 2.98 usec per loop

For the hashing function implementation, the proposal is to use Python standard
library's ``hmac`` [#hmac-python]_ module utilizing ``hashlib`` [#hashlib]_,
even though `cryptography <https://cryptography.io/en/latest/>`_ package is
available in Keystone. As ``hashlib`` demonstrates better performance
[#hazmat-vs-hashlib]_ for the :ref:`cryptographic hash algorithms proposed
<hash-algorithms-proposal>`.

.. code-block:: shell

   # hashlib.scrypt with work factor similar to the default configuration
   python -m timeit -n 1000000 -u usec \
     -s "import hmac" \
     "hmac.digest(b'secret_key', b'invalid_password', 'sha256')"
   1000000 loops, best of 5: 0.879 usec per loop

   # hashlib.scrypt with work factor similar to the default configuration
   python -m timeit -n 1000000 -u usec \
     -s "from cryptography.hazmat.primitives import hashes, hmac" \
     "h = hmac.HMAC(b'secret_key', hashes.SHA256()); h.update(b'invalid_password'); h.finalize()"
   1000000 loops, best of 5: 2.16 usec per loop

.. _other-deployer-impact:

Other Deployer Impact
---------------------

If the feature is not wanted by an operator, there is no impact.

If the feature is wanted, the following options are available in the
``security_compliance`` section:

* ``report_invalid_password_hash`` - to activate the feature.

* ``invalid_password_hash_secret_key`` - to apply HMAC secret key [#pepper]_
  when generating password hashes to make them unique and distinct from any
  other Keystone installations out there. Required, when feature is activated.

* ``invalid_password_hash_function`` - to define hash function to be used by
  HMAC.

* ``invalid_password_hash_max_chars`` - to configure number of characters of a
  hash to return. Would return full hash of invalid password when not
  configured (default). See also :ref:`security-impact`.

Developer Impact
----------------

None

.. _implementation:

Implementation
==============

Possible implementation - `932423: Support emitting partial hash of invalid
password <https://review.opendev.org/c/openstack/keystone/+/932423>`_

Assignee(s)
-----------

Primary assignee:
  bbobrov

Other contributors:
  None

Work Items
----------

TODO

Dependencies
============

None

Documentation Impact
====================

The options and the security impact need to be documented.

References
==========

* https://haveibeenpwned.com/API/v3#SearchingPwnedPasswordsByRange - a service
  to search for exposed passwords by their SHA-1 or NTLM hash prefix of 5
  characters - won't work for the hashes produced by the proposed
  implementation though due differences in hashing algorithm.

* https://opendev.org/openstack/keystoneauth/commit/2b305a718cb84edbdd977c26ca7e4134a3083c57,
  https://opendev.org/openstack/keystoneauth/commit/ccf6cb79033b2083d9177823094f7836eb68ae0d
  - ``keystoneauth`` hashes sensitive data in debug output with SHA256.

* https://compare-hashing-algorithms.mojoauth.com/hmac-sha256-vs-scrypt/ -
  HMAC-SHA256 vs scrypt

.. rubric:: Footnotes

.. [#owasp-cheat-sheet-peppering]
   https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#peppering
   - OWASP Cheat Sheet on peppering

.. [#hmac]
   https://en.wikipedia.org/wiki/HMAC

.. [#kdf-password-hashing]
   https://en.wikipedia.org/wiki/Key_derivation_function#Password_hashing

.. [#hashicorp-vault-audit-hashing]
   https://developer.hashicorp.com/vault/docs/audit#sensitive-information -
   HashiCorp Vault audit log hashes sensitive information with HMAC-SHA256.

.. [#pycadf-attachments]
   https://docs.openstack.org/pycadf/latest/specification/attachments.html

.. [#hmac-python]
   https://docs.python.org/3/library/hmac.html - python's Keyed-Hashing for
   Message Authentication

.. [#hashlib] https://docs.python.org/3/library/hashlib.html - python's Secure
   hashes and message digests

.. [#hazmat-vs-hashlib] https://github.com/pyca/cryptography/issues/6457 -
   hazmat hashes VS hashlib hashes

.. [#pepper] `<https://en.wikipedia.org/wiki/Pepper_(cryptography)>`_

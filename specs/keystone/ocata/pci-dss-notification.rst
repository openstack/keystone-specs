..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
PCI-DSS Notifications
=====================

Blueprint: `pci-dss-notifications <https://blueprints.launchpad.net/keystone/+spec/pci-dss-notifications>`_

Add reason field in notifications for various PCI-DSS events for auditing.

Problem Description
===================

Keystone currently does not include a reason for why a CADF notification
was sent for various PCI-DSS compliance events. For example, if:
``keystone.conf [security_compliance] lockout_failure_attempts``
is set to 5, and a user tries and fails to login 6 times, the notification for
the 6th attempt does not explain that the user has been locked out for failing
to login for the maximum number of attempts. Having this reason in the
notification would be beneficial for technical support and auditing purposes.

Proposed Change
===============

Add a reason code and reason type in the notifications for the following
compliance events:

PCI-DSS 8.1.6
^^^^^^^^^^^^^
Limit repeated access attempts by locking out the user ID after not more
than six attempts.

This will append the following reason code and reason type to the
existing ``identity.authenticate`` failure notification.

.. list-table::
   :widths: 45 250
   :header-rows: 1

   * - Reason Code
     - Reason Type / Message
   * - 401
     - Maximum number of <number> login attempts exceeded.

PCI-DSS 8.2.3
^^^^^^^^^^^^^
Passwords/passphrases must meet the following: (i) require a minimum length
of at least seven characters, and (ii) contain both numeric and alphabetic
characters. Alternatively, the passwords/passphrases must have complexity
and strength at least equivalent to the parameters specified above.

This will append the following reason code and reason type to the
existing ``identity.update.user`` failure notification.

``PATCH /v3/users/{user_id}``

``POST /v3/users/{user_id}/password``

.. list-table::
   :widths: 45 250
   :header-rows: 1

   * - Reason Code
     - Reason Type / Message
   * - 400
     - Password does not meet expected requirements: <regex_description>.

PCI-DSS 8.2.4
^^^^^^^^^^^^^
Change user passwords/passphrases at least once every 90 days.

This will create a new notification for:

``POST /v3/users/{user_id}/password``

.. list-table::
   :widths: 45 250
   :header-rows: 1

   * - Reason Code
     - Reason Type / Message
   * - 401
     - Password for <user> expired and must be changed

PCI-DSS 8.2.5
^^^^^^^^^^^^^
Do not allow an individual to submit a new password/passphrase that is the
same as any of the last four passwords/passphrases he or she has used.

This will append the following reason code and reason type to the
existing ``identity.authenticate`` failure notification.

``POST /v3/users/{user_id}/password``

.. list-table::
   :widths: 45 250
   :header-rows: 1

   * - Reason Code
     - Reason Type / Message
   * - 400
     - Changed password cannot be identical to the last <number> passwords.

Other
^^^^^
User attempting to change a password before a minimum password age elapsed.
This prevents users from erasing password history to re-use an old password.

This will create a new notification for:

``POST /v3/users/{user_id}/password``

.. list-table::
   :widths: 45 250
   :header-rows: 1

   * - Reason Code
     - Reason Type / Message
   * - 400
     - Cannot change password before minimum age <number> days is met.

Alternatives
------------

Events can be logged and parsed through log files.

Security Impact
---------------

None.

Notifications Impact
--------------------

A CADF notification should be emitted for each of the PCI-DSS events
triggered.  A sample notification would be:

.. code-block:: javascript

   {
       "priority": "INFO",
       "_unique_id": "3c030dc463114aa0ad17d703942b1a0e",
       "event_type": "identity.authenticate",
       "timestamp": "2016-10-07 01:30:53.075097",
       "publisher_id": "identity.controller",
       "payload": {
           "typeURI": "http://schemas.dmtf.org/cloud/audit/1.0/event",
           "initiator": {
               "typeURI": "service/security/account/user",
               "host": {
                   "address": "192.168.1.1"
               },
               "user_id": "6c3d3615c1fd4e868503f0f3f4366874",
               "id": "6c3d3615c1fd4e868503f0f3f4366874"
           },
           "target": {
               "typeURI": "service/security/account/user",
               "id": "a53ea0be-cb4b-529b-8648-09999df8f511"
           },
           "observer": {
               "typeURI": "service/security",
               "id": "9bdddeda6a0b451e9e0439646e532afd"
           },
           "eventType": "activity",
           "eventTime": "2016-10-07T01:30:53.072992+0000",
           "action": "authenticate",
           "outcome": "failure",
           "id": "272aad18-5fbe-580b-b39a-5f9c3ea42f79",
           "reason": {
              "reasonCode": "401",
              "reasonType": "Maximum number of X login attempts exceeded."
           }
       },
       "message_id": "e95a0285-ac25-43d6-b4d9-406997bba38c"
   }


Other End User Impact
---------------------

None. There will be no other end user impact.

Performance Impact
------------------

None. There will be no additional performance impact.

Other Deployer Impact
---------------------

None. There will be no other deployer impact.

Developer Impact
----------------

None. There will be no developer impact.

Implementation
==============

Primary assignee:

* gagehugo <gagehugo@gmail.com>

Other contributors:

* jaugustine <ja224e@att.com>
* lamt <tinlam@gmail.com>

Work Items
==========

* Add reason to notification for listed PCI-DSS events.
* Add unit tests.

Dependencies
============

This blueprint depends on the following:

`PCI-DSS blueprint <https://blueprints.launchpad.net/keystone/+spec/pci-dss>`_

Documentation Impact
====================

Notification structures outlined at http://docs.openstack.org/developer/keystone/event_notifications.html
will be updated to include the reason codes.

References
==========

`Midcycle Etherpad <https://etherpad.openstack.org/p/keystone-newton-midcycle>`_

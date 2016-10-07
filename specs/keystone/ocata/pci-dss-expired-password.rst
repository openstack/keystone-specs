..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
PCI-DSS Query Password Expired Users
====================================

Blueprint `pci-dss-query-password-expired-users <https://blueprints.launchpad.net/keystone/+spec/pci-dss-query-password-expired-users>`_

Problem Description
===================

Currently, when using the:
``keystone.conf [security_compliance] password_expires_days``
value, when a user's password expires and then must be reset by an
administrator, there is no way to query a list of users who are in
this state of password expiration. We would like the ability to retrieve
a list of users whose passwords has expired for technical support and
auditing purposes.

Proposed Change
===============

A new query will be added to the existing:
``GET /v3/users``
API call that would allow an administrator to query a list of users who are
currently locked-out due to password expiration. This will allow operators to
set up jobs to generate necessary audit lists and notifications.

**Query list of users based on their passwords' expiry time**

Gets a list of users based on their password expiry time.

.. code-block:: bash

   GET /v3/users?password_expires_at={operator}:{timestamp}

Where ``{timestamp}`` is a datetime in the format of ``YYYY-MM-DDTHH:mm:ssZ``
and ``{operator}`` can be either ``lt`` or ``gt``.  Note that
user can also do equality matching via
``/v3/users?password_expires_at={timestamp}``; however,
due to the nature of this query, it may not be as useful.

http://specs.openstack.org/openstack/api-wg/guidelines/pagination_filter_sort.html#filtering

Examples
========

**Query list of users whose password has expired before a given timestamp.**

.. code-block:: bash

   GET /v3/users?password_expires_at=lt:2016-10-10T15:30:22Z

**Response**

.. code-block:: json

   {
      "links": {
         "next": null,
         "previous": null,
         "self": "http://example.com/identity/v3/users"
      },
      "users": [
         {
            "domain_id": "default",
            "enabled": false,
            "id": "514a66612f53412796952414898a6b99",
            "name": "someuser1",
            "links": {
               "self": "http://example.com/identity/v3/users/514a66612f53412796952414898a6b99"
            },
            "password_expires_at": "2016-07-07T15:32:17.000000"
         },
         {
            "domain_id": "default",
            "enabled": true,
            "id": "ce8a21d43bc64ce6840346f0a14a7fa9",
            "name": "someuser4",
            "links": {
               "self": "http://example.com/identity/v3/users/ce8a21d43bc64ce6840346f0a14a7fa9"
            },
            "password_expires_at": "2016-10-09T00:21:04.000000"
         }
      ]
   }


**Query list of users whose password will expire after a given timestamp**

.. code-block:: bash

   GET /v3/users?password_expires_at=gt:2016-10-14T15:30:22Z

**Response**

.. code-block:: json

   {
      "links": {
         "next": null,
         "previous": null,
         "self": "http://example.com/identity/v3/users"
      },
      "users": [
         {
            "domain_id": "default",
            "enabled": false,
            "id": "514a66612f53412796952414898a6b99",
            "name": "someuser1",
            "links": {
               "self": "http://example.com/identity/v3/users/514a66612f53412796952414898a6b99"
            },
            "password_expires_at": "2016-10-17T15:32:17.000000"
         }
      ]
   }



Alternatives
------------

Operators can directly query the SQL backend for users whose password has
expired by checking the ``password_expires_at`` field.

Security Impact
---------------

None. The added API change has no additional security impact.

Notifications Impact
--------------------

No additional notification will be added for this query.

Other End User Impact
---------------------

None. There will be no additional end user impact.

Performance Impact
------------------

This call may fail if there is a very large number of users since pagination
is currently not supported.

Other Deployer Impact
---------------------

None. The added API change has no additional deployer impact.

Developer Impact
----------------

None. The added API change has no additional developer impact.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
   gagehugo <gagehugo@gmail.com>

Other contributors:
   lamt <tinlam@gmail.com>

Work Items
----------

* Implement new user query.
* Implement bindings in ``python-keystoneclient``.
* Implement unit tests.
* Document new user query usage.

Dependencies
============

This blueprint depends on the following:

* `PCI-DSS blueprint <https://blueprints.launchpad.net/keystone/+spec/pci-dss>`_

Documentation Impact
====================

Documentation in `api-ref` will be updated to include the added query
parameter and its usage.

References
==========

* `Midcycle Etherpad <https://etherpad.openstack.org/p/keystone-newton-midcycle>`_

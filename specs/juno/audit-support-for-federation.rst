..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Audit Support for Keystone Federation
=====================================

`bp audit-support-for-federation
<https://blueprints.launchpad.net/keystone/+spec/
audit-support-for-federation>`_

Keystone is expanding its support for federated identity to enable it to have
a more seamless integration into enterprise environments and to leverage
existing identity providers. The extra complexity associated with authentication
and authorization in federated Keystone deployments demands suitable audit
support to ensure the OpenStack environment is used in a compliant fashion. In
this blueprint we describe proposed auditing support for Keystone federated
identity operations using the `DMTF Cloud Auditing Data Federation (CADF) Open
Standard <http://www.dmtf.org/standards/cadf>`_, and leveraging `PyCADF
<http://docs.openstack.org/developer/pycadf/>`_.

Problem Description
===================
For Keystone, we need to define and implement support for new CADF audit event
records that capture the authentication and authorization behavior associated
with the mapping of federated attributes to group-based role assignments. Also,
for auditing purposes, we need to capture the user identity information
that is derived from external identity providers. This is crucial because in
the federated identity model for Keystone, user information is ephemeral and
no longer stored in Keystone directly.


Proposed Change
===============
Proposed change is to add new CADF based audit event notifications for
operations associated with federated identity. We intend to reuse the existing
notification work from `Icehouse release
<https://blueprints.launchpad.net/keystone/+spec/audit-event-record>`_.

The data within the notification will be comprised of information relating
to the federated user and the identity provider. Information such as:
username, userid, identity provider, protocol, and groups that the user was
mapped to, should be included within the notification.

For more details on what actions will trigger a notifications, please refer
to the `Notifications Impact`_ section of this specification.

Alternatives
------------
NONE

Data Model Impact
-----------------
NONE

REST API Impact
---------------
NONE

Security Impact
---------------
Auditing records need to be designed so that they do not accidentally publish
sensitive data, such as token information or passwords.

Notifications Impact
--------------------
New CADF notifications will need to be created for the following events:

* Federated user attempts to authenticate and retrieve an unscoped token.

* Federated user attempts to retrieve a list of projects.

* Federated user attempts to retrieve a list of domains.

* Federated user attempts to authenticate and retrieve a scoped token.

Other End User Impact
---------------------
NONE

Performance Impact
------------------
Performance should be marginally impacted if the CADF event notification for
federation support is enabled.

Other Deployer Impact
---------------------
NONE

Developer Impact
----------------
NONE


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* btopol

Other contributors:

* stevemar

* chungg

* mrutkows

Work Items
----------

* Define new audit event formats.

* Implement notifications for the new events.

* Create unit tests for the new events.

* Document new notifications events.


Dependencies
============

No new dependencies, however we will be using the PyCADF library,
which is already a required library for Keystone.


Testing
=======

Unit tests will be added for the new events.


Documentation Impact
====================

New audit events that have been added should be documented `here
<http://docs.openstack.org/developer/keystone/event_notifications.html>`_.


References
==========

* `DMTF Cloud Auditing Data Federation (CADF) Open Standard
  <http://www.dmtf.org/standards/cadf>`_

* `PyCADF library <http://docs.openstack.org/developer/pycadf>`_

* `Notification Documentation for Keystone
  <http://docs.openstack.org/developer/keystone/event_notifications.html>`_


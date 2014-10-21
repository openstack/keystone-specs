..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

===============
CADF Everywhere
===============

`bp cadf-everywhere
<https://blueprints.launchpad.net/keystone/+spec/cadf-everywhere>`_

As of the Juno release, Keystone emits two types of notifications. A simple
kind that is typically used as a callback function, that just has a resource
id as the only payload. The other is a CADF styled notification which contains
information that is useful for auditing.


Problem Description
===================

There has been more demand for cloud deployers to have the ability to audit
everything. This includes tasks such as project creation and deletion, as
well as role assignment creation and deletion. As such, we should start to
support these deployers by sending a notification with sufficient data to
perform a successful audit.

Proposed Change
===============

Create CADF notifications upon a `create`, `update`, and `deleted`
events for the following resources:

* group
* project (i.e. tenant)
* role
* user
* trust (immutable resource - no update notification)
* region
* endpoint
* service
* policy

Note: We will not be removing the simple notifications as they are useful for
callback functions, and there is already some support in Ceilometer for these
notifications. Barbican also relies on these as callbacks.

Alternatives
------------

Keep things as-is

Security Impact
---------------

None

Notifications Impact
--------------------

More CADF notifications

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* stevemar (Steve Martinelli <stevemar@ca.ibm.com>)

* topol (Brad Topol <topol@us.ibm.com>)

Work Items
----------

* Attempt to leverage the existing CADF decorators to create the payload for
  the notification and send it off.

* Possibly refactor the existing CADF decorator to be a bit more flexible.

Dependencies
============

None

Documentation Impact
====================

Update the notifications sections of documentation, possibly create a new
section for CADF notifications.

References
==========

`CADF Spec
<http://www.dmtf.org/sites/default/files/standards/documents/DSP0262_1.0.0.pdf>`_

`OS Profile for CADF
<http://www.dmtf.org/sites/default/files/standards/documents/DSP2038_1.0.0.pdf>`_

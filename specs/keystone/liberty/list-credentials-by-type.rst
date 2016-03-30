..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
List credentials by type
========================

`bp list-credentials-by-type
<https://blueprints.launchpad.net/keystone/+spec/list-credentials-by-type>`_

Currently the only attribute that you can filter a credential list by is
user_id. I want to be able to list by user_id and credential type (a required
field) so that I only get back my EC2 credentials (for example) when I do a
list.

Problem Description
===================

We can't filter credentials by ``type``, we just can list by user_id.

Proposed Change
===============

Add a new option ``type`` as a hint in the list credentials.

Alternatives
------------

None

Security Impact
---------------

None

Notifications Impact
--------------------

None

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

Assignee
-----------

Primary assignee:

Marianne Linhares  <mariannelinharesm>

Work Items
----------

* Add the hint `type` in the list credentials.

Dependencies
============

None

Documentation Impact
====================

The Identity API v3 Documentation must be updated according to these changes.

References
==========

http://eavesdrop.openstack.org/meetings/keystone/2015/keystone.2015-08-04-18.01.log.html

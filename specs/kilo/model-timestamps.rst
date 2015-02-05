..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
model-timestamps
================

`bp model-timestamps
<https://blueprints.launchpad.net/keystone/+spec/model-timestamps>`_

Currently Project, Domain, Role, User, Group models don't have a timestamp.
Although we can get a detailed auditing information via CADF audit event
records, we may need a simple way to determine when these resources were
created and when they were last modified.

We don't want to make one spec that fixes all models. So, this spec will only
implement timestamp for assignment models: Project, Domain, Role.

Because ldap assignment backend will be deprecated in next cycle and be removed
in the future, we only implement this in sql backend.

According to the reseller spec[2], the Domains and projects will become one
resource type, so this spec only implements timestamp for Project and Role.


Problem Description
===================

* For auditing, we may need a simple way to know how many projects were created
  or modified at a time interval.

* For administration, we may need a simple way to know when a project was
  created or last modified.


Proposed Change
===============

Actually, oslo.db has provided a TimestampMixin[1] object for timestamp models.
The only thing we need to do is to make Project, Role inherit from
TimestampMixin and then add a db migration script.


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

Assignee(s)
-----------

Primary assignee:

w-wanghong (wanghong <w.wanghong@huawei.com>)

Work Items
----------

* Update Project, Role model to inherit TimestampMixin

* Ensure timestamps are filtered before data is serialized to JSON

* Write db migration script

* Add tests


Dependencies
============

None

Documentation Impact
====================

Update the Projects and Roles sections of documentation.


References
==========

[1] ` oslo_db TimestampMixin
<https://github.com/openstack/oslo.db/blob/master/oslo_db/sqlalchemy/models.py#L115>`_

[2] ` reseller spec
<https://review.openstack.org/#/c/139824>`_

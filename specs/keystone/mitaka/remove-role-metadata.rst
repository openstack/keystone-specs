..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Remove role metadata structures
===============================

`bp remove-role-metadata <https://blueprints.launchpad.net/keystone/+spec/remove-role-metadata>`_


Remove the redundant "role metatdata structures" and the associated driver API
call.


Problem Description
===================

Part of the original assignment driver API for getting and setting the
assignments on a project/domain was to construct a metadata structure
that stored all the roles for a given project/domain in a dict. This probably
was derived from the early kvs implementation that stored assignments in this
form. Given that we have removed the kvs driver and now have a more
normalized storage of assignments (e.g. one row per assignment), this hang
over from the past has no advantage and just duplicates other paths for
retrieving assignments, leading to complicated code and maintenance issues.

Proposed Change
===============

It is proposed that this support in the assignment drivers is removed and the
two manager methods that use it (``get_roles_for_user_and_project`` and
``get_roles_for_user_and_domain``) call other existing driver methods to
extract the same information.

Alternatives
------------

None

Data Model Impact
-----------------

None

REST API Impact
---------------

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

* Henry Nash (henry-nash)

Work Items
----------

* Modify manager methods to call other driver methods
* Remove ``_get_metadata`` method from LDAP and SQL drivers.

Dependencies
============

For optimum performance this would capitalize on the filter performance driver
improvements already being undertaken, see:
`bp list-role-assignments-performance <https://blueprints.launchpad.net/keystone/+spec/list-role-assignments-performance>`)

Testing
=======

None

Documentation Impact
====================

None

References
==========

None

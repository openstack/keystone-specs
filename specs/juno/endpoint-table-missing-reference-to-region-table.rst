..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Add foreign key to region table in endpoint table
=================================================

`bp endpoint-table-is-missing-reference-to-region-table
<https://blueprints.launchpad.net/keystone/+spec/
endpoint-table-is-missing-reference-to-region-table>`_

Currently, Keystone has region table to model the region. The endpoint table is
still storing the region name in it, rather than a foreign key reference.

This proposal is to make the endpoint table to reference the region table for
the endpoint's region.

Problem Description
===================

Keystone has a ``region`` table for storing the region details, but it is not
consumed in the ``endpoint`` table, in which ``region`` column is still having
the name and not pointing to ``region_id`` as a foreign key.

Proposed Change
===============

This change will require a data migration to reconcile existing endpoint
records with records in the region table. The ``region`` column will be renamed
to ``region_id``. Existing region references in the ``endpoint`` table will
need to have corresponding records created in the ``region`` table.

For backwards-compatibility with clients that are not creating region records
prior to creating "regioned" endpoints, Keystone will need to automatically
create regions.

Deleting a region with associated endpoints will fail with a HTTP 403
Forbidden.

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
- kanagaraj-manickam

Other contributors:
- (none)

Work Items
----------

1. Add required database migration as described in the "Proposed changes."

2. Update the REST endpoint controller to take care of region as described in
   the "Proposed changes."

Dependencies
============

None

Documentation Impact
====================

None

References
==========

None

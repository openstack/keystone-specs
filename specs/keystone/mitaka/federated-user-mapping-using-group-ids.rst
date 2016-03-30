..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Direct users mapping using group ids
====================================

`bp federation-group-ids-mapped-without-domain-reference
<https://blueprints.launchpad.net/keystone/+spec/
federation-group-ids-mapped-without-domain-reference>`_

Allow user mapping using group ids without domain reference.

Problem Description
===================

Today, it's possible to provide a list of group names to Keystone via the
Identity Provider. However, a Domain must provided to map those groups. In the
eventuality of the Identity Provider having the reference to the group ids,
Keystone should be able to map those groups directly, without a domain
reference.

Proposed Change
===============

Keystone accepts group ids without any domain reference. The mapping should
include a new rule named ``group_ids``, and the list of group ids should be
provided by the Identity Provider.
Example of ``local`` rule specifying ``group_ids``:

::

    "local": [
        {
            "user": {
                "name": "{0}"
            },
        },
        {
            "group_ids": "{1}"
        }
    ]

As usual, an unscoped federated token will be issued.

Alternatives
------------

None.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Olivier Pilotte (opilotte)

Other assignees:


Work Items
----------

* Accepts Group IDs from the IdP without domain reference

Dependencies
============

None.

Documentation Impact
====================

All the changes must be reflected in the documentation.

References
==========

Accepts Group IDs from the IdP without domain -
https://review.openstack.org/#/c/210581/

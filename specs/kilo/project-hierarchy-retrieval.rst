..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
New query params to retrieve the project hierarchy
==================================================

`bp hierarchical-multitenancy-improvements
<https://blueprints.launchpad.net/keystone/+spec/hierarchical-multitenancy-improvements>`_

Now that we have the base implementation of hierarchical multitenancy in place,
we need to improve its robustness and usability by adding new features.

Problem Description
===================

In the first version, we implemented the basic features for Hierarchical
Multitenancy, which included the following:

* Create a project hierarchy, through the addition of the parent_id field in
  the project table;

* Get the project parents, subtree and full hierarchy;

* Delete projects without children (leaf projects);

* Grant/check/revoke role assignments that are inherited down the project
  hierarchy.

Adequate support to hierarchical multitenancy requires improvements such as
the following:

* Add a new format of a returned project hierarchy to better reflect the
  hierarchy (e.g., a hierarchy of dictionaries).

Proposed Change
===============

The currently implemented GET /projects/{project_id}?subtree_as_list and GET
/projects/{project_id}?parents_as_list calls return a nonstructured list
containing the projects in the hierarchy for which the caller has access to.
There are some use cases, for example, hierarchical quotas checking
(https://review.openstack.org/#/c/129420/), where the full hierarchy is needed,
but only the projects IDs are necessary and not the complete project
information. As the ID does not contain any sensitive information about the
project, it could be returned without any constraints.

There are two major steps we need to do in order to improve the current
implementation: the new GET /projects/{project_id}?subtree_ids and GET
/projects/{project_id}?parents_ids calls.

* The new GET /projects/{project_id}?subtree_ids call, which will return the
  subtree in the format of nested dictionaries, where every node will have
  the entries for its immediate children. Consider the following hierarchy::

    +------------------------+
    |           A            |
    |                        |
    |        /      \        |
    |                        |
    |       /        \       |
    |                        |
    |      B          C      |
    |                        |
    |    /   \       /  \    |
    |                        |
    |   /     \     /    \   |
    |                        |
    |  D       E   F      G  |
    +------------------------+


* For this hierarchy, when a user requests the subtree_ids from the project A,
  the following information will be returned::

        {
            "subtree": {
                "B": {
                    "D": null,
                    "E": null
                },
                "C": {
                    "F": null,
                    "G": null
                }
            }
        }


* The same will be done for the GET /projects/{project_id}?parents_ids call. So
  if the user requests the parents_ids for project D, the following would be
  returned::

        {
            "parents": {
                "B": {
                    "A": null
                }
            }
        }


* There will be a constraint that prevents the user from passing
  "subtree_as_list" and "subtree_ids" filters simultaneously, as well as
  "parents_as_list" and "parents_ids".


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

New API calls must be reflected on python-keystoneclient:

* GET subtree_ids
* GET parents_ids

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

New API calls will be available:

* GET /projects/{project_id}?subtree_ids
* GET /projects/{project_id}?parents_ids

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * Raildo Mascena raildo

Other contributors:
  * Rodrigo Duarte rodrigodsousa
  * Henrique Truta henrique-4
  * Samuel Medeiros samuelmdq
  * Adam Young ayoung


Work Items
----------

* Implement the new options to get the hierarchy: return the hierarchy using
  nested dictionaries.

Dependencies
============

None

Documentation Impact
====================

We must update the API Documentation (Identity API v3) according to these
changes.

References
==========

* `HM Kilo Summit <https://etherpad.openstack.org/p/hierarchical-multitenancy-kilo-summit>`_

* `Keystone Meetup Summit <https://etherpad.openstack.org/p/kilo-keystone-meetup>`_

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Materialized path - materialize-project-hierarchy
=================================================

`bp materialize-project-hierarchy <https://blueprints.launchpad.net/keystone/+spec/materialize-project-hierarchy>`_

To manage hierarchical multitenancy data effectively the existing adjacency
list paradigm is not enough: recursion is required to traverse the tree.
Materialized path concept allows you to find all ascendants, descendants,
children of the node in a single request operation.

Problem Description
===================

Given the project hierarchy as follows::

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

* request of all parents, grandparents and so on of the project will require
  recursion: the number of backend requests equals to tree depth;
* request of entire project's "family" will require recursion: to get sub-tree
  nodes it will be necessary to construct it from sub-sub trees an so on
  recursively;
* deleting project A will cause tree traversal accompanied by leaf-by-leaf
  deletion: find all sub-tree levels first and then delete required nodes in
  sequence from leaf to root.

Proposed Change
===============

It is possible to avoid recursion having entire project pedigree stored in the
model instead of simple parent link.

For example::

  DELIMITER = '->'
  project.pedigree = A.id + DELIMITER + B.id + .. + DELIMITER + D.id

So every project has it's pedigree stored that allows discovery of all projects
needing to be checked, and all descendant projects will have pedigree starting
with pedigree of ancestor.

This change is proposed to Resource SQL driver as entire HMT is implemented
this way:

* Add ``pedigree`` column to Resource model
* Change the logic of following driver methods to work with materialized path:
  * _get_children: to be removed
  * list_projects_in_subtree: all projects having pedigree starting with
  corresponding column value of project in question
  * list_project_parents: pedigree already contains all ancestry chain of id's
  and all corresponding project refs can be received with one bulk request.
  * is_leaf_project: check if project with pedigree starting with
  corresponding column value of project in question does not exist

In order materialized path to work backend must be capable of either prefixed
wildcard search or storing and indexing array fields. So ``pedigree`` column
may either be a text field containing delimiter-separated project id's or array
of them.

Alternatives
------------

`Pre-order Tree Traversal <http://en.wikipedia.org/wiki/Tree_traversal>`_
offers an interesting model allowing compact fields to be used, alas many tree
operations involve either complete tree reordering or, at least, comparison
operators performed not using indexes in SQL backends. The idea is to store
``left`` and ``right`` integer markers in each node following the rule: any
descendant node has ``left`` and ``right`` between ``left`` and ``right`` of
the current node.
(current.left < descendant.left < descendant.right < current.right)

`Adjacency list <http://en.wikipedia.org/wiki/Adjacency_list>`_ gives the most
flexible way to describe any graph, though in SQL backend it will result in
2-column indexed table of size up to N*(N-1)/2 that will lead to a poor
performance of modification operations.

`Adjacency matrix <http://en.wikipedia.org/wiki/Adjacency_matrix>`_ is a
perfect tool to perform a special graph theory calculations and requires
special software to be effective.

Caching of project structure may be an easier answer for small deployments and
when scalability becomes an issue the caching system itself can become a
complex solution with it's own issues.

Security Impact
---------------

none

Notifications Impact
--------------------

none

Other End User Impact
---------------------

none

Performance Impact
------------------

Operations on tree structure can be done in a single query without recursion.
For example:
* to delete a subtree one needs to delete projects with pedigree
starting with pedigree of the project in question;
* to get parent id's there is no need to use a backend at all - entire pedigree
is already stored in a model;
* to move a sub-tree to another node all moved nodes' ``pedigree`` has to be
updated so that the old parent id replaced with the new parent id.

Algorithms comparison::

  Get parent chain for D using ``parent_id``:
    1. Request parent for D -> B
    2. Request parent for B -> A

    Total: 2 requests (the deeper the tree the more requests are needed)

  Get parent chain for D using ``pedigree``:
    1. Disassemble ``pedigree`` -> A.id/B.id/D.id
    2. Request the content for A, B

    Total: 1 request (independent of tree depth)

  Get children of a node using ``parent_id``:
    1. Request all nodes with ``parent_id`` == node.id

    Total: 1 request, 1 filter by index

  Get children of a node using ``pedigree``:
    1. Request all nodes with ``pedigree`` starting with node.pedigree,
    size of node.pedigree + size of node.id + size of delimiter

    Total: 1 request, 1 filter by index, 1 sequence scan filter

  Get sub-tree of a node using ``parent_id``:
    1. For each node request all nodes with ``parent_id`` == node.id

    Total: 7 requests (1 for every node in sub-tree)
    BFS yields 2 requests here: 1 request per level of the tree.

  Get sub-tree of a node using ``pedigree``:
    1. Request all nodes with ``pedigree`` starting with node.pedigree

    Total: 1 request (independent of tree depth)

  Delete the sub-tree of the node using ``parent_id``::
    1. Traverse the tree to find leaf nodes
    2. Go up deleting them

    Total: 1 request per node on traversal + 1 per node on deletion
    May be optimised using BFS to iterate through levels and deleting groups
    of siblings.

  Delete the sub-tree of the node using ``pedigree``::
    1. Delete everything with ``pedigree`` starting with node ``pedigree``

    Total: 1 request (independent of tree depth)

  Move the sub-tree using ``parent_id``::
    1. update sub-tree top node's ``parent_id`` field with the new parent id

    Total: 1 request updating 1 node (independent of tree depth)

  Move the sub-tree using ``pedigree``::
    1. update ``pedigree`` of all nodes with ``pedigree`` starting with the old
    parent ``pedigree`` replacing it with the new parent ``pedigree``

    Total: 1 request updating all moved nodes

Other Deployer Impact
---------------------

Depth of the tree limitation may be increased if not removed at all: the single
restriction is a pedigree column capacity.

Developer Impact
----------------

Another gain is a possibility to have token validation in HMT case simplified.
To filter revocation events by token's project:

  1. Request pedigree for token's project
  2. Request if revocation event with project id in the pedigree exists

Example::

  User's U role R on project A was revoked. Client authenticated as user U has
  a token scoped to project D. Request to check if token is revoked: revocation
  event with user U, specified role, project in [A, B, D] exists.

Structure storage may feel a bit more complicated to developers to work with.

Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  amakarov

Other contributors:
  rodrigodsousa
  raildo

Work Items
----------

DB migration:
* Add `pedigree` column to the resource model
* Populate every project's `pedigree` column with actual pedigree calculated
using `parent_id`.
* Create btree index on `pedigree`.
* Drop `parent_id` with it's index.

Dependencies
============

* Include specific references to specs and/or blueprints in keystone, or in
  other projects, that this one either depends on or is related to.
* If this requires functionality of another project that is not currently used
  by Keystone (such as the glance v2 API when we previously only required v1),
  document that fact.
* Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?


Documentation Impact
====================

none

References
==========

* SQL implementation of adjacency list:
  `<http://en.wikipedia.org/wiki/Junction_table>`_
* Model Tree Structures with Materialized Paths in MongoDB:
  `<http://docs.mongodb.org/manual/tutorial/model-tree-structures-with-materialized-paths/>`_
* Materialized path description:
  `<http://www.ilias.de/docu/goto.php?target=wiki_1357_Materialized_Path&lang=en>`_
* The use-case description:
  `<http://dolphm.com/hierarchical-multitenancy/>`_

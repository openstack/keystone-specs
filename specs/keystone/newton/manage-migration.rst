..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Migration commands for rolling upgrades
=======================================

`bp manage-migration <https://blueprints.launchpad.net/keystone/+spec/manage-migration>`_

Provide a rolling upgrade steps in ``keystone-manage`` to allow for zero
downtime database upgrades.

Problem Description
===================

OpenStack services are moving towards support for rolling upgrades, with
today, nova, neutron and swift claiming such support. Other services are
working on this (e.g. glance). See the References section for more
information on support in other services. Prior to Newton, keystone did not
claim any such support, but we have decided to add this.

First, it is important to state that there are actually two intertwined
features when people talk about rolling upgrades;

1. The ability, in a multi-node service, to roll upgrades to nodes one at a
   time while the overall services remains in operation - so by definition
   allowing a mix of old and new nodes to run together at the same time using
   the same database.

2. Provide zero downtime for the service, including no temporary lockout of
   users during large database migrations while carrying out (1).

In Newton, we plan to support both (1) and (2). We have already initiated
support for rolling upgrades by only allowing additive changes to the database
(this is protected via tests).

To support just the rolling upgrade part, keystone would allow (given we have
only permitted additive database changes) the following sequence:

1. Take a keystone out of service and replace keystone with the new version,
   but don't run it yet. Execute the keystone-manage commands to upgrade the
   database. Wait and ensure all older nodes run fine. Only when you are happy
   with the situation, start processes running the new release and bring it
   online, one node at a time.

2. Upgrade the other nodes one at a time, by taking them out of whatever proxy
   configuration is being run, upgrading keystone and bringing them back on
   line. No database changes will take place during these steps since this was
   already done in (1).

In order to support such rolling upgrades, some migration scripts cannot leave
the database in the state they would ideally like. An example of this is where
a new attribute is added which has a default which needs to be written manually
by code (rather than a server default). If, during a rolling upgrade, new
entities are created via un-migrated nodes, such a new attribute will not get
the default. This causes a race condition between the process trying to clean
up such data and the processes creating it, and requires that the new release
of keystone understand both data schemas in order to function properly. See:
`Bug 1496500 <https://bugs.launchpad.net/keystone/+bug/1596500>`_.

Support for zero downtime upgrades without service interruption would require
keystone to support the gradual on-the-fly migration of data as they are
modified, with support for a final manual "migrate anything left" action. Such
on-the-fly support is implemented in the database object layer by Nova and
Neutron, which have support for looking in both the old and new locations for
data.

To solve that problem without writing and maintaining the necessary Python
code, we can instead rely on database triggers. Database triggers would give us
the ability to continue coding each release against a single schema, regardless
of the actual schema underlying the application (which must at least be a
superset of the schema the application knows about). More specifically, when
the previous release writes to the database schema it knows about, database
triggers would also update the new schema. When the next release writes to the
database schema it knows about, database triggers would also update the old
schema.

This approach allows us to eliminate:

1. The need to have the next release understand the previous release's schema
   (which would put a complex maintenance burden on developers, substantially
   raises the barrier to entry for new developers, and presents a risk of
   relatively subtle bugs).

2. The need for any clean up operations beyond a one-time data migration.

3. The need for the application to discover the state of the schema at runtime
   (a requirement which presents numerous possible race conditions).

Proposed Change
===============

It is proposed that we add new capabilities to ``keystone-manage`` which solve
the existing rolling upgrade problem and fit within the context of the full
rolling upgrades and zero downtime cross-project initiative.

There seems little or no commonality of the actual commands used by other
services' ``*-manage`` utility for the full zero downtime support. In order to
see how our proposed initial support fits within this full support, here are
the conceptual steps that will eventually be needed to upgrade a multi-node
configuration running code release X to X+1, where, for example, X+1 moves data
from one column to a new column:

1. One node has its code upgraded to X+1. The ``<service>-manage db_sync
   --expand`` command is run which creates new database structures, but does
   not migrate any data itself. It does, however, create database triggers to
   facilitate live migrating data between the two schema versions.

3. On the X+1 node, run ``<service>-manage db_sync --migrate`` to "forcefully"
   migrate all data from the old schema to the new schema. Database triggers
   will continue to maintain consistency between the old schema and new schema
   while nodes running the X release continue to write to the old schema.

4. Upgrade and restart all remaining nodes to the X+1 release one at a time.
   There will be a mix of releases writing to the database during this process,
   but database triggers will keep them in sync.

5. Once all the nodes are upgraded and writing to the new schema, remove the
   old schema and triggers using ``<service>-manage db_sync --contract``.

For keystone in this release, it is proposed that we support all the above
commands:

- ``keystone-manage db_sync --expand``: Expands the database schema by
  performing purely "additive" operations such as creating new columns,
  indexes, tables, and triggers. This can be run while all nodes are still on
  the X release.

  The new schema will begin to be populated by triggers while the X release
  continues to write to the old schema.

- ``keystone-manage db_sync --migrate``: Will perform on-the-fly data
  migrations from old schema to new schema, while all nodes serving requests
  are still running the X release.

- ``keystone-manage db_sync --contract``: Removes any old schema and triggers
  from the database once all nodes are running X+1.

The ``keystone-manage db_sync`` command (without options) will still be
supported, first to ensure that existing tooling and upgrade processes (which
do not try to execute a rolling upgrade) will continue to operate, and second
to provide a "force a database upgrade to completion" in case a deployer gets
into problems with a rolling upgrade. Once a ``keystone-manage db_sync``
command (without options) is executed, however, nodes running old code are no
longer supported. Running ``keystone-manage db_sync`` in this fashion will
execute all the phases (including the contract phase, if it is safe to do so),
and set the database migration status. This ensures subsequent rolling update
attempts at the next release are possible.

In terms of implementation, the ``keystone-manage db_sync --expand``,
``keystone-manage db_sync --migrate`` and ``keystone-manage db_sync
--contract`` phases will be driven by ``sqlalchemy-migrate`` repositories.

The proposed approach is designed to support both deployers who are upgrading
at major release cycles as well as those more closely tracking master.

One other aspect is that, in conjunction with services, we will not support
rolling upgrades across 2 releases. For example, once you are running Newton,
we will not support a rolling upgrade direct to the P release, you will need to
go to Ocata first.

Alternatives
------------

We could just use ``keystone-manage db_sync`` as the ``--expand`` step, but
since this would still want to print a reminder to run additional commands,
this would mean operators would not have a set of commands that did not print a
warning (which doesn't seem a good idea for production).

We could just use totally different ``keystone-manage`` commands, and not try
to make this fit the general trend for now::

    keystone-manage db_sync --initial-migration
    keystone-manage db_sync --complete-migration

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

Developers would need to be aware of the new database migration repositories,
and the requirements for each of them.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Henry Nash (henry-nash)

Additional assignees:
- Dolph Mathews (dolphm)
- Dave Chen (davechen)

Work Items
----------

1. Create three new database migration repositories (expand, migrate,
   contract).

2. Update the base ``keystone-manage db_sync`` (without options) to run any
   outstanding migrations from the legacy migration repository, all the
   ``--expand`` migrations, all the ``--migrate`` migrations, and then all the
   ``--contract`` migrations.

3. Implement the three new ``keystone-manage db_sync`` options, ``--expand``,
   ``--migrate``, and ``--contract`` to run their corresponding migration
   repositories.

4. Provide documentation to operators about the intended workflow.

Dependencies
============

None.

Documentation Impact
====================

Operator guides will need to updated.

References
==========

`Projects support rolling upgrades <https://governance.openstack.org/reference/tags/assert_supports-rolling-upgrade.html>`_

`Nova upgrade <http://docs.openstack.org/developer/nova/upgrade.html>`_

`Neutron upgrade <http://docs.openstack.org/developer/neutron/devref/upgrade.html>`_

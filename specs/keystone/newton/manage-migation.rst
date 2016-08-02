..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Migration completion after rolling upgrade
==========================================

`bp manage-migration <https://blueprints.launchpad.net/keystone/+spec/manage-migration>`_


Provide a "migration complete" step in keystone-manage to allow for any
database tidy-ups.


Problem Description
===================

OpenStack services are moving towards support for rolling upgrades, with
today, nova, neutron and swift claiming such support. Other services are
working on this (e.g. glance). See the References section for more
information on support in other services. Prior to Newton, keystone did not
claim any such support, but we have decided to add this.

First, it is important to state that there are actually two intertwined
features when people talk about rolling upgrades;

a) The ability, in a multi-node service, to roll upgrades to nodes one at a
time while the overall services remains in operation - so by definition
allowing a mix of old and new nodes to run together at the same time using
the same database.

b) Provide zero down-time for the service, including no temporary lockout of
users during large database migrations while carrying out a).

In Newton, we plan to support a), but not b), although will attempt to
minimize any temporary lockout (e.g. when database writes might be blocked
while data migration is happening). We initiated support for rolling upgrades
by only allowing additive changes to the database (this is protected via
tests).

To support just the rolling upgrade part, keystone would allow (given we have
only permitted additive database changes) the following sequence:

1) Take a keystone out of service and replace keystone with the new version,
but don't run it yet. Execute the keystone-manage command to sync the database.
Wait and ensure all older nodes run fine. Only when you are happy with the
situation, start the new keystone and bring it online.

2) Upgrade the other nodes one at a time, by taking them out of whatever proxy
configuration is being rnn, upgrading keystone and bringing them back on line.
No database changes will take place during these steps since this was already
done in 1).

In order to support such rolling upgrades, some migration scripts
cannot leave the database in the state they would ideally like. An example
of this is where a new attribute is added which has a default which needs
to be written manually by code (rather than a server default). If, during a
rolling upgrade, new entities are created via un-migrated nodes, such a new
attribute will not get the default. See:
<https://bugs.launchpad.net/keystone/+bug/1596500>`_

Support for zero down-time, would require keystone to support the gradual
on-the-fly migration of data as they are accessed, with support for a final
manual "migrate anything left" action. Such on-the-fly support is usually
implemented in the database object layer, which has support for looking in
both the old and new locations for data. It is anticipated that keystone will
eventually require this level of support.

Proposed Change
===============

It is proposed that we add to keystone-manage a new capability that just solves
the existing rolling upgrade problem, and yet fits within the context of the
full rolling upgrades and zero down-time cross project initiative. This
"migration complete" command, which an operator will run once all nodes have
had their code upgrade to the new version, and will tidy up the database
(e.g. write defaults to any new attributes that need them).

There seems little or no commonality of the actual commands used by other
services' manage utility for the full zero down-time support. In order to
see decide how our proposed initial support fits within this full support,
here are the conceptual steps that will eventually be needed to upgrade
a multi-node configuration running code release X to X+1, where X+1 moves
data from one column to a new column:

1) One node has its code upgraded to X+1 the
   *<service>-manage upgrade --expand* command is run which creates new
   database structures, but does not migrate any data. This command also sets
   the database status flag "migration_phase" to "Mixed X and X+1 nodes"
2) This node is restarted and we are now running a mix of X and X+1. X+1 code
   knows that if we are in a mix of X and X+1 nodes, then it must still read
   and write data to the old column location. If there are also new columns
   that have been introduced for new functionality (that X nodes know nothing
   about), then X+1 can write to these new columns.
3) The other nodes have their code updated one at a time
4) Once all the nodes are upgraded, set the "migration_phase" status to
   "X+1 nodes", using the *<service>-manage upgrade --rolling-upgrade-complete*
   command. Each node is then re-cycled one at a time. All code is now X+1.
   In this phase X+1 code will write to both old and new columns, and when
   reading will first look in the new column, then the old column, ensuring
   that X+1 nodes that have not been restarted yet still see data in the old
   locations.
5) Once all nodes have been restarted (so are aware of the "upgrade complete"
   status), then another status update is made
   (*<service>-manage upgrade --rolling-restart-complete*), at which point
   any node that sees this new status can now start migrating data on-the-fly.
   This migration means write to the new column only (although in an update
   the data may come from the old column). For read, you simply migrate the
   data to the new column before returning the entity.
6) At some point in the future (and before an upgrade to X+2), the
   *<service>-manage upgrade --force-migrate-complete* command is executed
   (perhaps with options for batching) that migrates any remaining data. (Note
   the actual command name for this phase varies wildly across services, the
   name given here is my own suggestion). On successful migration of all data,
   the status is automatically updated to mark that X+1 nodes no longer need to
   read or write to old locations. Restarting the nodes will bring this into
   effect (this is a performance optimzation, and no data integrity loss would
   occur if this was never carried out).
7) The process above is repeated for X+2. Once all X+1 nodes have been upgraded
   (i.e. the "migration_phase" will be "X+2 nodes") then the
   *<service>-manage upgrade --contract* command can be executed that
   will remove any columns/tables that wanted to be removed from X.

For keystone in this release, it is proposed that we support all the
above commands, with slight variation given that we are not yet supporting
on-the-fly migration:

*keystone-manage db_sync --expand* - this will, for Newton, both expand the
database and do the migration. We would need to document that phase might take
some time for large databases. It will also set the database status flag to
indicate a rolling upgrade is underway. This migration status flag will
be read by keystone on startup. For the Newton release, there is no difference
between upgrading from Liberty or Mitaka to Newton.

*keystone-manage db_sync --rolling-upgrade-complete* - this will mark the
code upgrade complete in a database status flag.

*keystone-manage db_sync --rolling-restart-complete* - this technically isn't
needed without on-the-fly migrations, but is included so that we get operators
used to the sequence.

*keystone-manage db_sync --force-migrate-complete* - this should be run once
all the nodes have been upgrade to the new release. Note that, for consistency
across installations, we want this to be run even if the upgrade was not
executed in a rolling fashion (i.e. even if this is a single node
configuration). Although not doing so would not cause any errors under normal
circumstances, it would be better to have all deployed databases at the same
migration level.

*keystone-manage db_sync --contract*. Typically this would need to wait until
all nodes are at X+2 (so Ocata), however for the current changes in Newton
(which consists of making one column non-nullable) it is OK to be run once
*db_sync --force-complete* has been run - and so we will also allow this.

When each command is run, it will check the migration status flag in the
database to ensure the command is being run at the appropriate point in the
migration sequence, and if not, will issue an error.

The *db_sync* command (without options) will still be supported, first to
ensure that existing tooling and upgrade processes (which do not try and
execute a rolling upgrade) will continue to operate, and second to provide
a "force a database upgrade to completion" in case a deployer gets into
problems with a rolling upgrade. Once a *db_sync* command (without options) is
executed, however, nodes running old code are no longer supported. Running
db_sync in this fashion will execute all the phases (including the contract
phase, if it is safe to do so), and set the database migration status. This
ensures subsequent rolling update attempts at the next release are possible.

One final new db_sync command will be provided
(*keystone-manage db_sync --status*), which will print out where we are in
the migration sequence (by reading the database status flag) and tell the
operator what the next step is.

In terms of implementation, the *db_sync --expand* and *db_sync --contract*
phases will be driven by sqlalchemy migration repos (the *db_sync --expand*
one is, of course, the existing migrate repo).  The *db_sync --force-complete*
phase is, in general, not suitable for a migrate repo since you want to allow
it to be run multiple times to batch the migration updates.

The proposed approach is designed to support both deployers who are upgrading
at major release cycles, as well as those more closely tracking master.

One other aspect is that, in conjunction with services, we will not support
rolling upgrades across 2 releases, i.e. once on Newton, we will not support
a rolling upgrade direct to the P release, you will need to go to O first. We
will, however, continue to support upgrading across 2 releases for the non
rolling upgrade approach (i.e. db_syns with no options).

Alternatives
------------

We could just use "db_sync" as the expand step, but since this would still want
to print a reminder to run migrate-force-complete, this would mean operators
would not have a set of commands that did not print a warning (which doesn't
seem a good idea for production).

We could just use totally different keystone-manage commands, and not try
and make this fit the general trend for now (given that we are not providing
zero down-time support), e.g.

*keystone-manage db_sync --initial-migration*
*keystone-manage db_sync --complete-migration*

We could use config value instead of a "migration_phase" database flag.
However, this would require pushing a new version of the config settings to
all nodes every time we wanted to change the value (as opposed to just bouncing
every node with a database flag), which seems to add additional complications
to the process.

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

Deployers would need to be aware of the new keystone-manage commands.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Henry Nash (henry-nash)

Work Items
----------

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

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Bootstrap via CLI
==================

`bp bootstrap <https://blueprints.launchpad.net/keystone/+spec/bootstrap>`_

Remove the ADMIN_TOKEN means of initializing a cluster with a CLI that has to
be executed on the same machine as the Keystone installation.


Problem Description
===================

`ADMIN_TOKEN` is a poor approach for initialzing a deployment.  It
provides a huge security risk for any site that fails to disable it
after initial deployment.  Since it is removed after the site is live,
there is no means to reenable it without A) restarting the service
and B) providing a huge surface for attack.  However, for a broken
system, sometimes it is the only tool that can effective fix things.

`ADMIN_TOKEN` is specified in the config file, which means that anyone
one with read access to the file has unlimited ability to affect
change in a keystone system. This is one of the values that forces the
config file to be reable only by root and the keystone service.  This
limits non-root users ability to read the config to determine the
state of the system and help troubleshoot.


Proposed Change
===============

Replace ADMIN_TOKEN with a set of CLI operations that affect the
necessarchanges to initialize a keystone server:

keystone-manage bootstrap

+---------------------------+-------+-----------------------------------------+
|Parameter                  |Default|Meaning                                  |
+===========================+=======+=========================================+
|bootstrap-username         |admin  |The username of the initial keystone user|
|                           |       | during bootstrap process.               |
+---------------------------+-------+-----------------------------------------+
|bootstrap-password         |None   |The bootstrap user password              |
+---------------------------+-------+-----------------------------------------+
|bootstrap-generate-password|None   |If set, will generate password           |
|                           |       |automatically and return it in the output|
+---------------------------+-------+-----------------------------------------+
|bootstrap-project-name     |admin  |The initial project created during the   |
|                           |       |keystone bootstrap process.              |
+---------------------------+-------+-----------------------------------------+
|bootstrap-role-name        |admin  |The initial role-name created during     |
|                           |       |the keystone bootstrap process.          |
+---------------------------+-------+-----------------------------------------+



Alternatives
------------

Direct database access, which would bypass all of the logic in the
system.

Precanned Database scripts, which would always put the system into a
known state; high risk of error and duplication, no way to fix a
wedged system.


Security Impact
---------------

Should reduce the attack surface of the Keystone server.  Anyone that
can read the config file can adffect these changes now.  With this
change, the user access would be limited to the same Unix users that
run the Keystone process, and would be managed via sudo.


Notifications Impact
--------------------

THe same notifications generated when these changes are made via the
API will be generated via this API.

Other End User Impact
---------------------

This will change how CMSs interact with Keystone.  The `ADMIN_TOKEN`
approach will be deprecated.

Performance Impact
------------------

None


Other Deployer Impact
---------------------

This will remove the ability to use `ADMIN_TOKEN` to troubleshoot, and
replace it with a more controlled approach.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------


Primary assignee:
  morganfainberg

Other contributors:
  ayoung

Work Items
----------


* Enabled bootsrap CLI
* deprecate ADMIN_TOKEN
* update devstack to use bootstrap
* remove admin_token from pipeline

Many releases later
* remove support for ADMIN_TOKEN

Dependencies
============

None

Documentation Impact
====================

Will change how all downstream project initialize Keystone.


References
==========

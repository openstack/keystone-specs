..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Deprecate keystone CLI
======================

`bp deprecate-cli
<https://blueprints.launchpad.net/python-keystoneclient/+spec/deprecate-cli>`_


The ``keystone`` CLI is superceded by the OpenStack unified CLI (OSC), so let's
get rid of it as soon as we can.


Problem Description
===================

The ``keystone`` CLI is superceded by OSC, as such it's a waste for us to
continue maintaining it. Since we can't just delete it we need to deprecate it
first.

Proposed Change
===============

The ``keystone`` CLI will print out Python's usual deprecation warning message
when it's run. By using Python's regular warnings module users can disable the
warning message, see https://docs.python.org/2/using/cmdline.html#cmdoption-W

The help text (``keystone --help``) will be updated to also say that the
command is deprecated.

This is part of a larger effort to get rid of the ``keystone`` command for
good.

Alternatives
------------

a) Engage in a major effort to fully support the ``keystone`` command,
   including implement all the identity V3 commands, duplicating the work of
   the OSC.

b) Make ``keystone`` a wrapper around the OSC.

Security Impact
---------------

The ``keystone`` CLI will only be patched for security and critical fixes since
it's deprecated.

Notifications Impact
--------------------

None. Notifications don't use the ``keystone`` CLI.

Other End User Impact
---------------------

Users will see a message every time unless they use -W to disable the warnings.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

Deployers will eventually need to move to using the unified CLI.

Developer Impact
----------------

Developers will need to move to using the unified CLI.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  <blk-u> Brant Knudson

Work Items
----------

* Change ``keystone`` CLI to print the deprecation warning when run.
* Change ``keystone`` help text to print that it's deprecated.
* Change ``keystone`` man page to say that it's deprecated.


Dependencies
============

None.

Documentation Impact
====================

The documentation should change all ``keystone`` commands to the equivalent
``openstack`` command.


References
==========

* https://docs.python.org/2/using/cmdline.html#cmdoption-W

* http://docs.openstack.org/developer/python-openstackclient/

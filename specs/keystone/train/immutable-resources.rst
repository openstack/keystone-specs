..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Immutable Resources
===================

`bug #1823258 <https://bugs.launchpad.net/keystone/+bug/1823258>`_

Keystone is a critical part of a cloud deployment. Administrators should take
special care with configuration changes in keystone to avoid cascading bad
changes to the rest of the cloud deployment, but keystone should also be more
robust against accidental footgunning.

Problem Description
===================

Keystone is responsible for many resources that are used through out other
services in an OpenStack deployment. For example, roles essentially map
permissions to a string that can be associated to a user via a role assignment.
Many roles are reused across OpenStack and some carry elevated authorization
needed to manage the deployment. In some cases, the accidental removal of a role
can be catastrophic to the deployment, since the deletion of a role triggers the
deletion of all role assignments any user has in any scope for that role. This
is particularly relevant to service users, which usually have an admin role
assignment, without which they cannot perform basic operations needed for the
cloud to be usable. The fix in such a case usually requires modifying database
entries by hand, which is a terrible practice in production environments.

Keystone should implement a more robust mechanism that allows operators to lock
specific resources, like important roles. A locked resource shouldn't be
deletable until it is unlocked, which adds a layer of protection for
deployment critical API resources, especially from accidental mishaps from the
command line or rogue/faulty administrator scripts.

Proposed Change
===============

Keystone roles, users, projects, and domains will gain an ``immutable``
flag as a `resource option`_. An immutable resource may not be deleted or
altered except to turn off the immutable flag.

For most resources, users will opt into locking these resources by setting the
flag. Eventually, the admin role should become immutable by default. However,
hardcoding immutability would be extremely backwards incompatible, so we
implement this in steps:

#. Add an ``immutable`` resource option to the role model. This will be off by
   default, always.

#. Add an opt-in flag ``--immutable-roles`` to the ``keystone-manage bootstrap``
   command which sets the ``immutable`` resource option on the default roles
   (``admin``, ``member``, ``reader``) to ``true``. The command should also log
   a warning that this will become default behavior in the future if they do not
   set it.

#. Add a ``keystone-status`` check to alert operators if they have not made the
   default roles immutable.

#. Change the ``keystone-manage bootstrap`` behavior to make roles immutable by
   default and opt-out available with ``--no-immutable-roles``.

.. _resource option: https://docs.openstack.org/keystone/latest/admin/resource-options.html

Alternatives
------------

* Make role more like domains, which must be disabled and then deleted. This is
  not backwards compatible.

* Change roles to be soft-deleted. This doesn't change the fact that role
  assignments are hard-deleted, but could make it easier to recover since the
  role ID still resides in the database.

* Change horizon to give a visual alert for potentially destructive actions like
  deleting the admin role. This doesn't protect against bad scripts.

Security Impact
---------------

This doesn't have a direct security impact.

Notifications Impact
--------------------

No notifications will be admitted with this new resource option.

Other End User Impact
---------------------

Administrative users will need to unset the ``immutable`` flag for a role if
they truly want to delete or alter the role. Client changes will be needed to
allow the adminitrator to set a role as immutable or mutable. Non-administrative
end users should see no difference.

Performance Impact
------------------

Negligable performance impact. Keystone have to determine if resources are
locked during writeable operations and act accordingly, which would be
considered business-specific logic.

Other Deployer Impact
---------------------

Deployers will be alerted to opt into the new behavior. Detailed upgrade
instructions will be made available in the release notes and the command help
output.

Developer Impact
----------------

Developers of custom role backends will need to be aware of the new property.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Colleen Murphy (cmurphy)

Work Items
----------

Outlined in `Proposed Change`_.

Dependencies
============

Requires `resource options scaffolding`_ to be implemented for the role driver.

.. _resource options scaffolding: https://review.openstack.org/624162

Documentation Impact
====================

This will require an update to the API reference and instructions in the
administrator guide for upgrading and managing roles.

References
==========

* `Meeting discussion <http://eavesdrop.openstack.org/meetings/keystone/2018/keystone.2018-12-04-16.00.log.html#l-172>`_

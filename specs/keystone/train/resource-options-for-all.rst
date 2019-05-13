..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Resource options for all resource types
=======================================

`bug #1807751 <https://bugs.launchpad.net/keystone/+bug/1807751>`_


Resource Options have been implemented for the User resource type. These are
used for PCI-DSS controls (e.g. exempting a user from password change
requirements) and Multi-Factor Auth login rules. The other resource types
within keystone will benefit from a similar set of technologies. Examples
of use cases are:

* Limit login to specific origins (IP Addrs) for tokens scoped to a given
  project or domain

* Apply default PCI-DSS options to all users contained within a Domain, e.g.
  exempt all service users in a ``service`` domain from password change
  requirements.

* Apply default Multi-Factor-Auth rules to all logins scoping to a given
  domain or project.


Problem Description
===================

Each resource type may have explicit controls or options that are unique to
that resource class (e.g. exempting a user from PCI-DSS password change
requirements). This specification proposes expanding the resource option
functionality from users to encompass all resource types. This is implementing
the scaffolding for future options to be built upon. No options will be
added in the scope of this specification.

This is being added to support concepts such as immutable resources, defaults
for entire domains related to PCI-DSS options (e.g. exempting all service users
from password expiry), default Multi-Factor-Auth rules for scoping to a given
domain, etc.

Proposed Change
===============

Add the same controls, db tables, and responses for all resource types that
currently do not have resource options implemented. This will be implemented
as part of the base SQL Model class defined within keystone, all future
resource types will be expected to implement the resource option functionality.

Alternatives
------------

* Option #1: implement resource options explicitly when needed for a
  given resource class.

* Option #2: do not implement resource options and tool the functionality
  separately for the same behaviors.


Security Impact
---------------

This is structure and code capability implementation and should have no
security impact.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Users will see an added ``resource_options`` response for resources.

Performance Impact
------------------

The additional DB lookups for extracting the resource options will add
additional load to keystone. Most resources will have no resource options
and those that do have resource options will be (by default) leaning on
SQL indexes to mitigate the potential additional load.

Other Deployer Impact
---------------------

No deployer impact until future options are implemented.

Developer Impact
----------------

Developers will need to implement resource option functionality for all new
resource types after this spec is implemented.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  morgan fainberg <morgan.fainberg@gmail.com>


Work Items
----------

* Implement DB Migrations to add the resource option tables for each resource
  type/class.

* Implement the API handlers to process and validate resource options for each
  resource type.

* Implement resource option base code into the SQL Model base defined within
  keystone.


Dependencies
============

None


Documentation Impact
====================

Documentation on adding new resource options will be needed.


References
==========

N/A
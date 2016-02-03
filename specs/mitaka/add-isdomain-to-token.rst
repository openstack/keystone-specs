..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Add is_domain to the token for projects acting as a domain
==========================================================

`bp add-isdomain-to-token <https://blueprints.launchpad.net/keystone/+spec/add-isdomain-to-token>`_


For project scoped tokens that are issued for a project acting as a domain,
the token should also contain is_domain=true.


Problem Description
===================

In policy files today, the creation of concepts such as "domain admin" rely
on the use of domain scoped tokens. Support for domain tokens doesn't really
exist outside keystone (even in horizon), which is problematic. We have
already discussed (and rejected) dual scoped tokens (i.e. a token which
contains both a project ID and domain ID, both set to the same ID). However,
the problem of how we enable policy protected domain related operations in
clients of keystone remains.

Proposed Change
===============

This proposal is to simply add is_domain=true to any token that is issued
for a project that is acting as a domain. With that change, it would then
be possible to write policy rules that today require a domain token, to
instead work with regular project tokens, while still providing the same
level of protection. For example, the a typical policy rule of::

"identity:create_user": "rule:admin_required and domain_id:%(user.domain_id)s"

would be replaced with::

"identity:create_user": "rule:admin_required and project_id:%(user.domain_id)s and is_domain:True"

In a similar way, this would allow a nova policy file to specify that, for
instance, servers were not allowed to be created in a project acting as a
domain::

"compute:create": "role:vm-manager and is_domain:False"

This simple change would allow the entire policy file to be re-written to never
require domain scoped tokens, paving the way for their eventual removal.

In order for a project token to be used in this way, it must, of course, be
possible to request a project scoped token for a project acting as a domain.
The current issue with this is the potential ambiguity of requesting this
by project name, since there could be a clash between the name of the project
acting as a domain and a project somewhere in that domain. It is proposed that
when request such a token by project name:

* There is no change to the current scope request definition
* If the project name is truly unique, then that's the project you get
* If there is a clash between domain and a project, you always get the project
* If, in such a clashing situation, you really do want a project scoped token
  to the project acting as the domain, then you need to either use ID or rename
  either the domain or the project first.

Such a proposal for requesting a token leaves open for the future the
possibility of more comprehensive options such as passing in a hierarchy
definition if, for example, we wanted to move to support only requiring a
project name to be unique within its immediate parent.

Alternatives
------------

A number of alternatives have already been discussed in other proposals, and so
far have been rejected.

Data Model Impact
-----------------

None

REST API Impact
---------------

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

To take advantage of this, deployers would have to change their policy files.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
    henry-nash

Work Items
----------

- Modify token generation checking code to handle name clashes
- Add the attribute to the token
- Update the v3sample policy file to match

Dependencies
============

None

Testing
=======

None, beyond the regular unit testing.

Documentation Impact
====================

Changes to the Identity API and configuration.rst.

References
==========

None

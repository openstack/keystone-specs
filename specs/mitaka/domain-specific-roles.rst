..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Domain Specific Roles
=====================

`bp domain-specific-roles <https://blueprints.launchpad.net/keystone/+spec/domain-specific-roles>`_

Extended the concept of Implied Roles to allow the prior role to be
domain-specific, allowing a domain administrator to create roles that are
meaningful to their users. These roles are just management roles, and expand
out into their implied roles for token purposes.

Problem Description
===================

The existing definition of a role is a direct link to the policy rules, i.e.
the role name is the item that is referenced directly from policy. This will be
referred to as a "policy role" in this blueprint.

The Implied Roles specification creates the ability where a prior role can be
defined that expands out into a set of implied roles at token validation time.
Both the prior role and all the implied roles end up in the token - i.e. all of
them are policy roles.

`spec implied-roles <https://review.openstack.org/#/c/125704/>`_

Policy files that reference policy roles are, and should be, tightly
controlled in any cloud. An error in one of these files could be very
damaging. As such, they are not likely to be changed too often. The modeling
of a set of human-understandable roles that might be given to users, however,
may change quite often. In public clouds (or multi-customer clouds in general),
for instance, the cloud provider will most likely publish their "role model",
listing what roles customer admins should assign their users to be able to
issue given APIs. In order to service the greatest number of differing
customers, this role model is likely to become much more granular than it is
today (since, of course, the role model is common to all their customers).

For a specific customer, however, it is unlikely that whatever set of granular
roles a cloud provider chooses will be meaningful or represent a relevant
grouping of roles that will suit their particular usage model. Ideally a domain
administrator would want to create roles that are meaningful to the particular
users of that domain and somehow map these onto the role model defined by the
cloud provider. These domain-specific roles would be private to their domain
and exist only in that domain's namespace, since they may make no sense to
users in other domains. Furthermore these domain-specific roles should not show
up in any tokens - only the roles that they map to (which are policy roles that
are part of the cloud provider's roles model) should appear in tokens.

The Implied Roles capability provides the mapping described above, with the
domain specific role being a prior role. However, as currently described,
it requires that prior roles are global in scope (i.e. not private to the
domain) and that prior roles are also inserted into the token, hence the above
is not possible.

Proposed Change
===============

It is proposed that we build on Implied Roles capability to allow the creation
of domain specific prior roles. Domain specific prior roles (which are simply
role entities with a new domain_id attribute) can be used to define implied
roles (i.e. by implying a set of policy roles), with the only difference being
(compared to a regular prior role) that when the effective roles are calculated
for the token, the domain specific prior role itself will not be included.

A domain specific role can be used (in terms of assignment APIs) everywhere
that a global role can be used today. Given that at token generation, domain
specific roles will be expanded into their policy roles, there will be
no impact on services outside of Keystone (and no change to policy files). For
fernet tokens (which don't contain the roles anyway), they will be expanded at
token validation time. Furthermore, domain specific roles can be used with
federation, allowing any of the attested users from IdPs, that are trusted by
the domain, to gain domain specific role assignments via the existing federated
mapping to Keystone groups.

The domain_id attribute of a role is immutable, you can't flip a role between
being global and domain specific.

Alternatives
------------

Provide the ability to have domain-specific policy files. This would be a very
complex undertaking.

Data Model Impact
-----------------

For SQL, the roles table will be expanded to support domain specific roles.

REST API Impact
---------------

Modifications to the role API to allow creation of domain specific roles.

Security Impact
---------------

Any change to the role or policy model has the potential to affect security.
The proposal doesn't, however, significantly increase the attack vectors, since
in essence it is simply providing a more convenient alternative to manually
adding a set of existing global policy roles to users.

Notifications Impact
--------------------

Any existing notification regarding roles will be extended to include
domain specific roles.

Other End User Impact
---------------------

None

Performance Impact
------------------

The main impact would be at token generation and validation time, but this
will be not different to that of implied roles in general.

Other Deployer Impact
---------------------

None

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

- Add manager/driver support for domain specific roles
- Add controller for domain specific roles
- Add keystoneclient library support for domain specific roles
- Add openstack cli support for domain specific roles

Dependencies
============

`spec implied-roles <https://review.openstack.org/#/c/125704/>`_

Testing
=======

None

Documentation Impact
====================

Changes to user documentation to describe new API.

References
==========

None

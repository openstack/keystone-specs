..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Split-up Assignments, making the Role-Assignment piece pluggable
================================================================

`bp pluggable-assignment <https://blueprints.launchpad.net/keystone/+spec/pluggable-assignment>`_


Modify the assignment component by splitting out (and making pluggable) the
parts that relate to actual assignments, in order to allow alternative
assignment solutions to be provided.


Problem Description
===================

The assignment controller and backend are really misnomers - there are
effectively the "rest-of-identity", that was left over when we split out the
actual identity piece (i.e. users and groups). Within this "assignment"
component we have the store for domains, projects, roles....and the actual
assignments themselves. Domains, projects and roles are the lingua franca
used throughout OpenStack for policy control to APIs. The actual assignment
relationships between these entities (which are stored in Keystone) are not
shared outside of Keystone - rather these assignments are used to determine
which roles are included in a token for a given user for a given scope of
target entity (i.e. project or domain).  Hence in reality what we call
the "assignment" component of Keystone today, consists of:

1. Storage and APIs for defining domains, projects and roles.
2. Storage and APIs for defining assignment relationships.
3. A set of APIs that evaluate the assignments, in order that the correct set
   of roles can be placed in a scoped token.

The fact that the above are all lumped together in one component leads to a
number of issues.

First, the component is overly complex. This has been shown to be true in terms
of number of new defects being found in this area recently. Further we want to
make significant changes in this area: e.g. hierarchical projects and
restructuring of the list_role_assignents() method for performance reasons.
These changes can only make things worse.

Second, for developers and cloud providers that would like to offer alternative
types of assignments (e.g. ABAC, enhanced relationship models etc.), all they
really need to provide is their own version of 2 (above) and honor the API of
3. However, since the "assignments" backend contains all of the three items
above, it is not simple to provide just the actual assignments piece. The
advantage of enabling alternative access control models in this way is that the
rest of OpenStack will continue to work - i.e. no change to token formats or
policy files in other projects. The OpenStack community as a whole will benefit
if we can let Keystone be a platform through which new forms of access control
can be brought into play, with the market eventually deciding which of those
best matches customer requirements.

Proposed Change
===============

This change would split the current "assignment" controller and backend into
two:

* resources - containing domains, projects and roles
* assignments - containing current assignment model

This change will both simplify the support for the core entities on which
the rest of OpenStack depends, while making it easier to normalize the
various (slightly different) manager/driver methods for listing assignment
relationships, e.g.:

* list_role_assignments
* list_grants
* get_roles_for_groups
* get_roles_for_user_and_domain
* list_projects_for_user
* list_domains_for_groups

This is particularly important when inheritance is considered - there is a
general lack of conformity of how assignment inheritance is handled across
the various flavors of driver calls - that is currently leading to a number
of defects.

A further side benefit of this change is that deployers could choose different
backend technologies (e.g. LDAP vs SQL) for resource entities and assignments
that match better the scale & update profiles of the respective entities.

Making this split will not, in and of itself, have any functional change, but
will enable more reliable support for the current capabilities, while providing
a better base to support the changes in-flight, such as hierarchical
multi-tenancy and role assignment performance improvements. It also provides
a platform that enables the experimentation of new assignment models, while
ensuring their expression of assignments can be still be used by the rest
of OpenStack.

Alternatives
------------

None

Data Model Impact
-----------------

Other than splitting the data models into two backends, the only change is
that assignments would no longer use a foreign key to track the role id.
Although this does, or course, reduce referential integrity, this is
considered a low risk change given that we have already removed the foreign
keys for the user, group, domain and project objects.

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

There is likely to be a mild performance impact from this change for list
assignment API calls that return the refs of projects, domains or roles since
we can no longer execute the extraction of assignment ids and their subsequent
turning to refs in one go. The impact of this will be mitigated by supporting
bulk loads of refs from a list of ids in the new base entity drivers. See the
references section for details of early WIP code that shows how this would
be done.

Other Deployer Impact
---------------------

Deployers would be able to set the backends for resources and assignments
separately, although by default they would be the same (to keep with existing
configuration settings).

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

- Split the controller and backends
- Split up the test cases into relevant files
- Provide a sample alternative controller and backend
- Document how to provide a new assignment controller and backend

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Changes to the configuration.rst and developing.rst

References
==========

An early WIP of what would be a first phase of implementing this bp is
available at: https://review.openstack.org/#/c/130954/

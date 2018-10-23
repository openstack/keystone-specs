..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Explicit Domain IDs
===================

`Explicit Domain IDs <https://bugs.launchpad.net/keystone/+bug/1794527>`_

Problem Description
===================

Many organizations are deploying multiple OpenStack deployments. There
are many reasons that these systems are not shared into on large cluster.
Some are owned by different sub organizations. Others are kept separate
due to application life cycle. Some are physically isolated from each other.
All of these issues prevent co-operation across clusters.

If the organization uses LDAP for the user backend, the user entry in
the id_mapping table gets a new local_id generated as a hash of the
domain_id and the unique Identifier from the LDAP server.  If the
Domain ID is different in two different deployments, the user ends up
with two distinct user IDs.  An attempt to build an Audit train across
multiple deployments then has to correlate the user IDs across
different Keystone servers.

To avoid this, operators want to keep the LDAP users in a domain with
a consistent ID across deployments.  The only domain with a
predictable ID, however, is `default.` The default domain is used
during the initial install, and thus has service users. Thus, it is
impractical to put the LDAP users in the default domain. If a site
wished to have two LDAP servers in domain specific backends, only one
could be put in default.

Today, if a deployer desires to keep two Keystone servers in sync
while  avoiding Database replication, they must hack the database to
update the Domain ID so that entries match. Domain ID is then used for
LDAP mapped IDs, and if they don't match, the user IDs are
different. It should be possible to add a domain with an explicit ID,
so that the two servers can match User IDs.

An additional effort is underway to make the Federated Identity
sources use the same mechanism as LDAP. This effort is hampered
by the same domain_id restrictions as LDAP.

Making the identifiers consistent across multiple deployments will aid
in several use cases:

It will make it easier to synchronize role assignments across two
distinct deployments, as the user ID from the first can be used in the
second.

Applications will now be able to predict what user-id a user would
have in a cluster, even before that user visits the cluster. This
allows the system to pre-create user records, and have them link to
the identity source when the user does finally authenticate to the
Keystone server.

One structure that can be used for multiple deployments is for each
Keystone server to represent a different region, and for each region
to have a set of domains for which it is the system of record. This
allows local writes and avoids conflicts. In order for a central
system to keep track of the domain IDs, and to synchronize them across
different servers, one of the systems needs to be able to explicitly
assign them.

Proposed Change
===============

When creating a domain, the user can pass the optional parameter
`explicit_domain_id` to be used when creating the domain.  A domain
created this way will not use an auto-generated ID, but will use the id
passed in instead.

Identifiers passed in this way must conform to the existing ID
generation scheme:  UUID4 without dashes. Note that this API will only be
accessible via system membership, restricting it to deployment administrators
and operators.

Alternatives
------------

Galera Sync of the database between sites.  This will only scale to a
small number of locations.  With local HA requirements of 3 nodes per
site, this will likely scale to a single digit number of locations.

Make Domain names immutable and change the scheme to use the names.
This will break all the existing LDAP deployments, as well as break
API backwards compatibility.

K2K Federation has been discussed as a way to help manage multiple
sites, but it does not address the need to make the User ID consistent
for Audit purposes.  In addition, K2K adds an additional layer of SAML
indirection, without providing additional value here:  it solves the
problem where users are stored in one Keystone server, but need access
to a separate cloud.  As such, K2K will suffer from all the same
problems as any of Federated Identity Provider.

Security Impact
---------------

This should have little to no direct security impact.  It will,
however, greatly aid in auditability, as user IDs can be correlated
across multiple deployments.

There is little risk of a domain administrator abusing this new option.
Creating a Domain is a rare operation, reserved for Cloud Admins.  As such,
they are keeping domain IDs in sync to aid their own organizational goals.
If they chose to not sync a domain, it will only affect their own cluster, and
not the others.  The degenerate case for Auditing will be the current state.


The IDs for users will now fall into the category of
"predictable-but-not-settable."  Since the uuid is a hash of the
string, and not explicitly setable, the will not be a potential for
"User_ID squatting." where a user pre-allocates an entry to block
another user.

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

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * Adam Young <ayoung>

Work Items
----------

Change to Keystone Server
Change to Openstack SDK
Change to OpenStack CLI

Dependencies
============

None

Documentation Impact
====================

New option needs to be in documentation, including an update to the
LDAP and Federation docs.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Cross-backend IDs for User and Group Entities
=============================================

`bp multi-backend-uuids
<https://blueprints.launchpad.net/keystone/+spec/multi-backend-uuids>`_

Enable Keystone to generate and maintain unique IDs for user and group entities
that are keystone-installation-wide, irrespective of whether the entity is
local to Keystone, in LDAP or federated.


Problem Description
===================

Originally Keystone was responsible for creating all its user and group
entities, along with assigning them a unique ID.  With the advent of LDAP and
federation, this is no longer the case.  However, the Keystone API still
assumes it can uniquely identify a user and group from just the ID provided.
Further, for Openstack installations that support multiple customers, where
each customer is represented as a domain with its own LDAP, it is simply
impractical to just search for the entity across all backends.

What is required is for Keystone to continue to be able to provide an ID via
its API for all entities it encounters (however they were created) and to be
able to find that entity again efficiently when it receives an API call to
manipulate that entity using the ID the caller provided.  Clients of Keystone
should be unaware of any such changes below the API.

A key requirement is that the introduction of any such support should not
invalidate or change any IDs already issued by Keystone in existing
installations, since such IDs may be persistently stored by clients of
Keystone (e.g. Swift).

Proposed Change
===============

The proposal is that Keystone will build an identity mapping of *Public IDs*
to underlying backend identifiers, so that Keystone can route API calls to the
correct driver.  Clients of Keystone will only see the Public ID and the
generation of the Public ID will happen on-the-fly within Keystone.  This
change was originally proposed (and was close to being integrated) for
the IceHouse release, but by mutual agreement was deferred to allow discussion
of this topic at the Juno summit, which endorsed the general direction
outlined in this proposal.

This mapping is only relevant to the identity component of Keystone (i.e.
users and groups) and  will be implemented as a pluggable mapping backend,
with the only storage backend initially provided as SQL.

As was stated in the summary, one of the main drivers for this requirement is
the support of domain specific backends (e.g. LDAP) for Openstack installations
that support multiple customers (or even multiple divisions with an
enterprise).  The framework for such support was added to the Keystone codebase
in the Havana release within the identity/core.py module.  This support is
marked as experimental, however, since it does not support a concept of
ID mapping - and instead tried to deduce the domain scope of the API from
the context of the call (e.g. to which domain is the token scoped).  While
this works in many cases, it is an incomplete solution - examples of where
it cannot deduce the correct domain include:

- Authentication via user ID
- Cross-domain role assignment

This change therefore builds on the multi-domain framework in the current
codebase, but removes the "domain deduction" code in favor of ID mapping.

There are a number of design items that need to be taken into account in
implementing this change:

- Ensuring compatibility with existing entity IDs.
  There are two kinds of user and group entity IDs in Keystone installations
  today:

  - UUID IDs for SQL backed identities, or for those entities created by
    Keystone in a RW LDAP backend (the uuid is stored in an LDAP attribute)

  - Various LDAP attributes as selected by operators (e.g. email address) for
    LDAP backed identities created outside of Keystone

  Since (ignoring the current experimental support for domain-specific
  backends) all IDs can only be related to the single chosen SQL or LDAP
  identity driver, upgrading to Juno with multi-backend UUIDs should not affect
  these IDs that have already been exposed. The proposal is that by default no
  identity mapping entries will be created, since there is only the single
  standard identity driver. The only case where entity IDs from the standard
  identity driver will be mapped is if the cloud provider wants to ensure that
  the whatever LDAP attribute is being used as the entity ID is not exposed as
  thePublic ID.  This is achieved by setting the ``backward_compatible_ids``
  configuration option to ``False``.

- Public ID generator algorithm.
  While the obvious choice is simply another regular uuid4 hex string, it is
  proposed to use a hash algorithm so that Public IDs are regeneratable
  from the underlying local identifier components. This has a number of
  advantages:

  - Loss of entire mapping table.
    If the mapping table was ever lost, then an additional recovery mechanism
    (as opposed to reverting to a backup) would be to simply allow the table
    to regenerate on-the-fly as user and group entities were encountered by
    Keystone. It should be noted, however, that if the assignments table was
    also lost, then such a facility could only be part of recovery.

  - Ease of purging mapping table.
    In the case where the identities are managed outside of Keystone (e.g. LDAP
    or federation), then it is likely that stale entries representing users
    and groups that have been deleted from the underlying identity store will
    remain stored in the mapping table.  Whilst benign, there will clearly be
    a need for some kind of periodic cleanup.  If the table was regeneratable
    then simply purging the entire table (or all entries for a given domain)
    using an option in keystone-manage would be a simple cure (albeit
    incurring a slight performance degradation each time a given entity was
    next encountered).  In addition, an option will be provided to
    delete individual entries based on the local entity information.

  - Ensuring consistent Public ID generation when using multiple Keystones.
    For some large installations, multiple Keystones may be running, backed
    by a replicated database. In such circumstances it is possible that two
    different Keystones might encounter an entity for the first time and
    both create a mapping for it (due to delays in database replication). In
    the regeneratable case, they would both create the same Public ID and
    avoid a table row clash.  While it should be noted that this issue
    already exists in the regular RW SQL backend (for example, two users
    of the same name could be created with different UUIDs via two different
    Keystones), the case where a RO LDAP backend is providing the identity
    store via a mapping is far more likely to generate a clash (since it only
    requires two requests to read the same entity at roughly the same time
    while running multiple Keystones).

  The selection of a hash algorithm to use is a balance between resulting key
  size (and hence column width of any client that will need to store the
  generated ID), probability of collision and security of the encoded data.
  It should be noted that for most installations, the data being hashed
  here (local ID and domain ID) is not likely to considered secret - hence the
  actual security of the hash involved is less important. However, to
  provide flexibility across all use cases, a pluggable generator backend
  will be implemented, with a sha256 generator provided as the default. The
  key produced by generators will be limited to a maximum string size 64 bytes
  (and any generator producing more than that will cause an exception to be
  thrown).

  There are also several complexities that are created by creating an
  identity mapping layer:

  - In the codebase today the controllers generate the UUID ID for an entity
    they are creating, and pass this to the manager/driver layer. All our unit
    testing of the manager/driver layer also assumes that the caller specifies
    the ID. Now that the manager layer is dynamically building a Public ID
    mapping table, it seems inappropriate that the controller layer thinks it
    is in control of ID generation. The proposal is, therefore, to remove
    the ID generation from the User and Group controllers, and let the
    identity manager carry out this role. Note that this only affects the
    controller-manager interface, not the driver interface itself.  This
    change, while only affecting a few lines of production code, will result
    in a large set of mechanical changes to our unit tests.

  - If the underlying driver supports UUIDs (for example the current SQL
    backend), then there seems little advantage in taking a hash of the
    UUID. The proposal therefore is to use the local UUID as the Public ID
    for such drivers, and load a null mapping into the mapping table.

Alternatives
------------

An alternative approach was discussed during IceHouse development and at the
Juno summit of creating the mapping within the ID itself - i.e. encoding all
the details needed to find the entity in the backend in the ID string.  One
such proposal was:

<local ID>@@<domain-name>

The problem with this proposal is that, as it stands today, both domain name
and any local ID can both be 64 bytes long - and the entity ID Keystone needs
to return is also just 64 bytes.  A discussion on the dev list explored the
option of increasing the size of the entity ID being returned by Keystone,
which resulted in strong objections to this from other projects (that are
consumers of these entity IDs). The discussion thread can be found here:

https://www.mail-archive.com/openstack-dev@lists.openstack.org/msg17506.html

The above solution was, in fact, prototyped during IceHouse development as part
of the development of this change and can be found here:

https://review.openstack.org/#/c/74214/14

A further refinement to this alternative idea was also discussed in terms of
compressing the number of bytes required for the domain info, while also
restricting the number of bytes allocated to the local ID, so as to fit the
while identifier within the currently spec of 64 bytes.  While this may work
in many practical cases, the general consensus is that this would restrict us
unduly in terms of what information we could store to uniquely identify the
entity in question in its local backend.  A simple example is that some
backend stores may store user and group IDs in different namespaces, and
hence the mapping should also store what type of entity this is.  Further,
for Federation, there may be additional information we might wish to store.

If in the future we did want to implement some kind of scheme along these
lines, then the currently proposed architecture for this change would allow
a mapping backend to be implemented that simply provided the encoding to
and from the Public ID rather than actually storing the mapping attributes
in a table.

An earlier version of this proposal suggested providing an option for
the algorithm for Public ID generation, e.g. choosing between a regular
UUID and a hash. This was dropped as there was little advantage in the use
of regular UUIDs.

Data Model Impact
-----------------

The data model changes involve the creation of a new table that provides
the mapping:

.. code-block:: python

    class IDMapping(sql.ModelBase, sql.ModelDictMixin):
        __tablename__ = 'id_mapping'
        public_id = sql.Column(sql.String(64), primary_key=True)
        domain_id = sql.Column(sql.String(64), nullable=False)
        local_id = sql.Column(sql.String(64), nullable=False)
        type = sql.Column(
            sql.Enum(map.EntityType.USER, map.EntityType.GROUP,
                     name='type'),
            nullable=False)
        __table_args__ = (sql.UniqueConstraint('domain_id', 'local_id', 'type'),
                          {})

The unique constraint is defined to ensure two mappings to the same
Public ID cannot be stored in the table.

No further indexes are suggested, since (except for keystone-manage) there
are only two real access patterns:  By Public ID (which is the prime key) and
by specifying all the local identifiers (which should already be indexed due to
the unique constraint).  Given that keystone-manage is unlikely to be
time-critical, the trade-off of further indexes is unlikely to be worth it.

REST API Impact
---------------

There are no new API calls for this proposal.  Some existing API calls have
added restrictions placed upon them.

- "List users" and "List groups"
  In the case when Keystone is configured for domain-specific backends (via
  the configuration file) these API calls require a domain scope to be
  specified. This can be done explicitly by using the already supported
  domain_id filter or implicitly by using a domain scoped token. If neither of
  these are provided, the call will return a 401 (Unauthorized) error code.

- "Add user to group"
  Since group membership is considered a function of identity and the
  underlying driver backend, membership across different domain-specific
  backends is not supported and will return a 403 (Forbidden) error code.
  This does not affect role assignment across domains and backends which
  remains unrestricted.

Further, if an unsupported identity mapping generator algorithm has been
specified in the Keystone configuration file, then any identity API is likely
to generate a 500 (Internal Server Error) return code.

Security Impact
---------------

The identity mapping function described in this proposal does not store user
data - it simply maps a Public ID to the local identifier.  Although in general
the local ID (even for LDAP) is not considered sensitive data, one benefit of
this proposal is that the LDAP local ID does not escape Keystone (since only
the Public ID is exposed).  This is in contrast to the current single-domain
LDAP implementation that exposes the local ID defined by the LDAP driver as the
publicly visible entity ID.

Notifications Impact
--------------------

The existing identity notification will continue to function, although this
references the Public ID rather than any local identifier information.

Other End User Impact
---------------------

None

Performance Impact
------------------

The introduction of a mapping layer will obviously have some impact. However,
since the mapping layer is only used, by default, in domain-specific backend
situations which are not supported in production today, there will be no impact
on a default or existing installation.  When it is used the following
additional database calls will be made:

- A non-PK table lookup for each entity returned by a "List" call
  for users and groups (in order to map to the Public ID)
- A mapping entry table is created the first time a user or group item is
  encountered (to create the mapping)
- A PK lookup for every user and group API used to manipulate an entity (to
  lookup the mapping)

Only the first of these has any chance of having any noticeable performance
impact. If this proves to be the case, then the optimization listed above in
the section of ID generation using hashing could be implemented in a follow-on
patch.

Other Deployer Impact
---------------------

The two main impacts on a deployer will be:

- Choice of Public ID generator algorithm
  This was discussed earlier in this specification.

- Periodic purging of stale entries from the mapping tables
  As also described earlier, for backend entity stores that are managed outside
  of Keystone, in general there is no reliable notification mechanism that
  Keystone could use to auto-purge stale entries from the mapping table. To
  enable manual purging, keystone-manage will support a new option
  ``mapping_purge`` which will allow the operator to specify the following
  options:

  - ``keystone-manage mapping_purge --all``--
    This will purge all mappings
  - ``keystone-manage mapping_purge --domain-name <name>``--
    This will purge all mappings for the named domain
  - ``keystone-manage mapping_purge --domain-name <name> --local-id <ID> --type <user|group>``--
    This will purge the mapping for the named local identifier
  - ``keystone-manage mapping_purge --public-id <ID>``--
    This will purge the mapping for the named public ID

Developer Impact
----------------

There are no changes to the identity driver interface with this proposal,
although there are two changes to the controller-manager interface:

- The (now unused) optional parameter ``domain_scope`` will be removed (this
  was the "domain deduction" from the earlier Havana implementation.

- The explicit ``user_id`` and ``group_id`` parameters in the Create call
  for those entities, since the manager layer will now generate the ID.  The
  manager will will then pass this ID to the driver layer.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  henry-nash

Work Items
----------

The set of items required are:

- Removal of the "domain deduction" parameter from the identity controller-
  manager interface.

- Ensure a domain is either explicitly or implicitly defined for the List
  user and group entities in the controller.

- Removal of the ``user_id`` and ``group_id`` parameters from the identity
  manager Create calls for those entities.

- Implementation of the backend mapping layer.

- Provide the two ID generators, UUID and hash, controller by a configuration
  option.

- Modify the identity manager layer to call the identity mapping layer to
  ensure only Public IDs are exposed to the controller.

- Modify ``keystone-manage`` to provide options for purging the mappings.

- Ammend the existing ldap backend unit testing to cover the cases of
  backward compatible and non backward compatible IDs as well as to provide
  better coverage for the multi-backend scenarios.

- Provide specific unit testing for the identity mapping layer.

- Modify existing unit testing that calls the Create user and group APIs
  to support the manager generating the ID.

A full implementation that matches the above spec is already available at:

https://review.openstack.org/#/c/74214/


Dependencies
============

None


Testing
=======

No additional tempest testing is proposed since the existing tests are
sufficient to catch potential anomalies in the Public ID.


Documentation Impact
====================

Since there is no change to the API, the only documentation changes required
are to the configuration guide.


References
==========

1. Juno Etherpad: https://etherpad.openstack.org/p/juno-keystone-user-ids

2. `NIST high level policy information on hashing algorithms and use
   <http://csrc.nist.gov/groups/ST/hash/policy.html>`_

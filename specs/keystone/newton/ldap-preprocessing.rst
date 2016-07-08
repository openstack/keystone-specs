..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
LDAP preprocessing
==================

`bp ldap-preprocessing <https://blueprints.launchpad.net/keystone/+spec/ldap-preprocessing>`_


Keystone can use LDAP as an identity backend. LDAP attributes probably are
not uuid4, and we need our entities have uuid4 ids. To perform
ldap_attribute <-> keystone_id mapping, there is id_mapping_api. However,
it is very slow when there are many identities.


Problem Description
===================

id_mapping_api is a mapping layer between LDAP entities and keystone entities.
When an operation on an entity needs to be performed, id_mapping_api changes
the id from public (uuid4) to local (LDAP-specific) or vise versa.

It does that by looking up public and local id for each entity. For N users in
LDAP it makes N SELECT queries to the database. Even worse, if M entities
don't yet have a public id, id_mapping_api performs M INSERT queries. All of
this happens when a user makes an API call to keystone. When N and M are big,
the call will result in timeout.

The problem with N SELECT queries can be worked around by selecting all rows
from the mapping table and performing the lookup for public or local id in
memory. However, the problem with M INSERT queries cannot be worked around so
easily.

One might want to use bulk insert. However, race condition might happen in this
case::

                               +
                  Process 1    |    Process 2
                        +-------------+
    Select entities #1, #2, #3 | Select entity #2
    from the table; 0 returned | from the table; 0 returned
    since they don't exist yet | since it does not exist yet
                        +-------------+
                               | Generate public id for #2
                               | Insert public_id for #2
                               |
                        +-------------+
    Generate public_id for     |
    #1, #2, #3. Bulk insert    |
    for all items.             |
                               |
                               v

At this point process 1 will fail because there is already an entry
about entity #2. It cannot be ignored, because the id, assigned in process 2,
was already returned to the user.


Proposed Change
===============

Implement new command for `keystone-manage`: `prepare_ldap`. It will:
1. Fetch all users from LDAP
2. For all users create a mapping

The command will be executed by operators, when they know that LDAP has many
entries.

Running the command **is not required** for integrating LDAP. It will provide
a better user and operator experience for cases when there are many entities
in LDAP.

Alternatives
------------

Only implement a workaround for N SELECT queries.

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

The workaround with N queries it will decrease the number of SQL queries to
constant number and will require O(N) additional memory.

`keystone-manage prepare_ldap` command will be ran by operators. It will not
be exposed to the users. It will speed-up the first call or the call after
many new LDAP users were added.

Other Deployer Impact
---------------------

Deployers will have an ability to run the command right after configuring LDAP.

It is not a requirement to run it. No existing deployment scripts will be
affected.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bbobrov (breton)

Dependencies
============

No dependencies

Documentation Impact
====================

Document the new command.

References
==========

* https://review.openstack.org/339294

* https://github.com/openstack/keystone/blob/master/keystone/identity/core.py#L569

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Expiring Group Memberships Through Mapping Rules
================================================

`bug #1809116 <https://bugs.launchpad.net/keystone/+bug/1809116>`_

Add federated users to the groups that they receive from the mapping rules.
This membership is only carried by the token and not persisted in the
database. The membership expires, but can be renewed when the user
authenticates with the same group.


Problem Description
===================

Currently, federated users that receive their authorization as part of
mapping to a group, cannot create trusts or application credentials. Creation
will fail saying that the user doesn't have the role. That is because that
role assignment is only valid for the duration of their token, and not
permanently added to the user.

We cannot make the group membership concrete, because that user would then
have those permissions even if their state in the external identity provider
changes.

The problem has been reported and discussed as `bug 1589993
<https://bugs.launchpad.net/keystone/+bug/1589993>`_.


Proposed Change
===============

Architecture
------------

When a federated user authenticates through the mapping driver, the list of
group IDs is added to the token. We will persist that group membership to the
database.

Every time the user authenticates through federation, the list of groups is
reevaluated, and already existing (expired or not) memberships are renewed
and new ones are added.

Each group membership is individually expirable and renewable.

Expiring group memberships will only be used for user group memberships
through the mapping driver and not through other drivers or backends.

The user will be allowed to create application credentials. however they will
be unable to authenticate with them after the expiring group membership which
they depend on have expired.

The longevity of the membership will be dependent on the identity provider by
which the user authenticates with. When setting up an identity provider in
keystone, the cloud admin will be able to configure this setting on a per-idp
basis.

Implementation
--------------

A new model `ExpiringUserGroupMembership` will be created with a corresponding
sql table.

  .. code-block:: python

    _tablename__ = 'expiring_user_group_membership'
    user_id = sql.Column(sql.String(64),
                         sql.ForeignKey('user.id'),
                         primary_key=True)
    group_id = sql.Column(sql.String(64),
                          sql.ForeignKey('group.id'),
                          primary_key=True)
    idp_id = sql.Column(sql.String(64),
                        sql.ForeignKey('identity_provider.id'),
                        primary_key=True)
    last_verified = (sql.Date, nullable=False)


`last_verified` which will store the time of the last authentication of the
user from the identity provider with the appropriate membership. This field
will not be updated when the user is logging in through any other means than
through the identity provider.

Likewise, the identity provider model will be extended to include a new field
`authorization_ttl`. This will be the default, for existing identity provider
or when creating new identity providers when a custom one is not specified.

If `current_time > last_active + authorization_ttl` then the group membership
will expire and the user will be unable to use authorization dependent on it.

The `/v3/users/{user_id}/groups` API will be extended to returns expiring
group membership alongside their expiry time.

Alternatives
------------

Instead of expiring the group membership, expire (disable) the entire user.
This would result in the same end-effect if users can only come from one
identity provider. This however would prevent further plans for "linked
accounts" and users having different levels of access based on their method
of authentication.

Another alternative is to persist the group membership and role assignments in
the application credential, and not add them to the user itself. Then, force
the expiry and renewal on the application credential object. This was the
initial proposal prior to discussions with the keystone team and other
stakeholders.

Security Impact
---------------

Since Keystone doesn't have access to the external identity provider to get
notified when a users permissions are revoked, there will be a lag between
when a user has their permissions revoked and their group membership expiry.
During that time they will be able to use the application credential, but will
be unable to renew it.

Notifications Impact
--------------------

A notification will be emitted when a user authenticates using federation and
expiring group memberships are created. Notifications will also be emitted
when expiring group memberships are renewed and when they expire without
being renewed.

Other End User Impact
---------------------

Application credentials and trusts will be usable by federated users.
Changes to `openstackclient` and Python clients.

Performance Impact
------------------

Authenticating through an external identity provider will generate extra load,
as keystone will automatically check and renew group memberships.

Other Deployer Impact
---------------------

 * New configuration option for `default_authorization_ttl`.


Developer Impact
----------------

None


Implementation
==============

Assignee
--------
 * Kristi Nikolla <knikolla>


Work Items
----------

 * Add configuration options
 * Extend database model and write migrations
 * Extend API
 * Write documentation and release notes


Dependencies
============

None


Documentation Impact
====================

New documentation for the feature.


References
==========

 * https://bugs.launchpad.net/keystone/+bug/1589993
 * https://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/application-credentials.html

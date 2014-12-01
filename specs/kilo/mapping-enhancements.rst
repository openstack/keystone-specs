..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Enhance Federation mapping algorithms
=====================================

`bp mapping-enhancements
<https://blueprints.launchpad.net/keystone/+spec/mapping-enhancements>`_


Federated authentication and authorization was shipped with OpenStack Icehouse
release.  One of the core features is the ``mapping engine`` that translates
identity attributes from federation (e.g. ``SAML2/OpenId Connect``) assertions
into Keystone specific parameters (``user_id``, ``groups``). The mapping engine
operates on top of ``mapping rules`` configured for every registered
``Identity Provider``. Mapping rules specify how the assertions should be
interpreted to grant ephemeral users access to certain resources. However, lots
of discussions arose around enhancing the mapping engine and the mapping rules
'language' for easier and more powerful configuration capabilities. It is also
extremely important to allow cloud administrators be in power to control
authorization grants assigned to ephemeral (federated) users.

Problem Description
===================

Currently, part of the infrastructure preparation is configuration of
``projects``, ``domains``, ``role assignments`` and ``groups`` for ephemeral
users. This approach is good enough, however for some specific use-cases it
could be made more transparent and automated. In order to cover all possible
combinations, lots of rules need to be created. For instance, there is a need
to create separate rules for mapping members of remote group ``devs`` into
local group ``devs``, members of remote group ``admins`` into local group
``admins`` and so on.

Proposed Change
===============

Extend the mapping engine to allow passing groups shipped in an assertion/claim
to be subset of effective local Keystone Groups.  That is, by being a member of
groups ``group1``, ``group2`` and ``group3`` in user's home company/institute,
a user would effectively become a member of such groups as an ephemeral user in
a cloud he is bursting into. Note that cloud administrators would still need to
add such groups a priori. It is extremely important to be able to whitelist and
blacklist the effective list of groups.  Otherwise, by assigning a group
``admin`` in user's home configuration, he would become a group ``admin``
member in the cloud.  By having whitelisting and blacklisting functions, cloud
administrators would still keep power to control users access. Current mapping
engine capabilities, like specifying effective groups should stay, so the
proposed change is additive and backward compatible.  Also, the cloud
administrator (or whoever builds mapping rules) should be able to specify
the effective domains (identified either by ``name`` or ``id``).  This is
required, as groups will be usually specified by names, and the domain name
must be present to precisely identify the group entity in the system.

Proposed changes include:

* Change ``local`` entities, so the effective groups can be identified by
  ``name`` and ``domain``, instead of ``id`` only. An example with old and new
  sytax of local ``groups`` identification would look like:

.. code-block:: javascript

    {
        "group" {
            "name": "developers",
            "domain": {
                "name": "clients"
            }
        }
    },
    {
        "group": {
            "id": "89678b"
        }
    }

* Indicate parameter as a list of groups that should be mapped directly and add
  another ``keywords`` for whitelisting and blacklisting the effective list of
  groups.  In the following example, effectively only the intersection of ';'
  delimited set of groups from ``ADFS_GROUPS`` and ``whitelist`` (allow this
  set of groups) as well as difference between ``ADFS_GROUPS_2`` and
  ``blacklist`` (allow all except this list) would be passed.
  Only one keyword (``whitelist`` or ``blacklist``) can be used in a rule.
  If both are used, Keystone will reject such set of mapping rules.
  Example:

.. code-block:: javascript

    {
        "remote": [
            {
                "type": "ADFS_GROUPS",
                "whitelist": [
                    "g1", "g2", "g3", "g4"
                ]
            },
            {
                "type": "ADFS_GROUPS_2",
                "blacklist": [
                    "admin", "superadmin", "managers"
                ]
            },
        ]
    },
    {
        "local": [
            {
                "groups": {0},
                "domain": {
                    "name": "domain_name"
                }
            },
            {
                "groups": {1},
                "domain": {
                    "id": "456hy643"
                }
            },
        ]

    }


Alternatives
------------

None.

Security Impact
---------------


* Does this change touch sensitive data such as tokens, keys, or user data?

  No.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

  No.

* Does this change involve cryptography or hashing?

  No.

* Does this change require the use of sudo or any elevated privileges?

  No.

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

  It adds new functionalities to a mechanism that is already in Keystone.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  No.


Notifications Impact
--------------------

None.

Other End User Impact
---------------------

``python-keystoneclient`` does not require any changes as it's the rules
structure that is changed.
Lack of preparing all effective groups may ease overall configuration.


Performance Impact
------------------

None.

Other Deployer Impact
---------------------

If using direct groups mapping, deployers should carefully specify whitelists
and blacklists so no privilege escalation is possible.


Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Marek Denis <marek-denis>

Other contributors:
    Nathan Kinder <nkinder>
    Rodrigo Duarte <rodrigodsousa>
    Victor Silva <vsilva>
    Henry Nash <henry-nash>

Work Items
----------

* Implement ``get_group_by_name`` methods allowing for fetching ``group``
  object identified by a ``name`` and ``domain``. This method would not be
  exposed via v3 Identity API.

* Enhance mapping engine so the group can be identified by ``name`` and its
  ``domain``.

* Add keywords ``whitelist``, ``blacklist`` as well as the ability to treat an
  assertion parameter as a collection of groups a user is a member of.

Dependencies
============

None.


Documentation Impact
====================

New mapping capabilities should be carefully explained and documented, pointing
out possible security risks if the cloud is misconfigured.


References
==========

None.

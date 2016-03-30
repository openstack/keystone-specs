..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Federated Service Providers in Keystone
=======================================

`bp k2k-service-providers
<https://blueprints.launchpad.net/keystone/+spec/k2k-service-providers>`_

This specification describes main steps required for reworking
Keystone2Keystone federation shipped in Juno release to improve user experience
and avoid breaking Keystone's architecture.

Problem Description
===================

Keystone2Keystone federation delivered in Juno is currently marked as
``experimental`` and happens to miss few important points. Remote services are
marked as regions in the Service Catalog, however they cannot be accessed with
a token issued by local Identity Service. Apart from that not all required URLs
are specified, forcing client to know them apriori. Therefore there is a need
for re-architecting few aspects:

* ``regions`` should not be used for indicating remote service a client can
  burst into.
* a client should be able to fetch all required information needed for bursting
  into remote clouds (for instance ``authURL`` as well as urls specific for a
  SAML2 authentication workflow.)

Proposed Change
===============

Keystone should be enhanced with new set of objects called ``Service Provider``
(``/v3/OS-FEDERATION/service_providers/``) where a trusted Service Providers
are being configured. Information stored within such object should include
information like:

* ``authURL`` - an url where client can get his token once he has authenticated
  via SAML2 federated protocol. Example:
  ``https://keystone.example.com:5000/v3/OS-FEDERATION/identity_providers/<idp>/protocols/<protocol>/auth``.
* a specific url - usually a dedicated url where assertions are being
  sent. Example: ``https://keystone.example.com:5000/Shibboleth.sso/POST``.

Each ``Service Provider`` should be identified by a user specified system
unique name, like already existing within Keystone ecosystem ``Identity
Providers``. ``Identity API`` should be enhanced with 5 new ``Service
Provider`` operations:

* Create
* Delete
* Get
* List
* Update

Apart from that, Service Catalog should be extended with a new entry -
``service_providers``.  Users willing to burst into remote clouds would query
that entry in the Service Catalog. Optionally, proper filtering of the
``Service Providers`` on a per user basis could be added (e.g. ``userA``
can burst into ``cloud1`` and ``cloud2`` whereas ``userB`` can burst into
``cloud2`` and ``cloud3``. Those constraints should be reflected in the Service
Catalog proposed for each of the users).

As Keystone2Keystone federation was marked as experimental in the Juno release,
a script/procedure for migrating service providers configured as ``regions`` to
a first class ``service provider`` objects will not be provided.

Alternatives
------------

* Keep using regions as remote endpoint where users can burst into, however
  this would presume users know apriori at least ``authURL`` of the remote
  services as well federated protocol to be used.

* Accept SAML2 Service Provider Metadata file as an input required for creating
  ``Service Provider`` objects. This means the user would upload Service
  Provider Metadata as a request body while creating and updating information
  about ``Service Providers``. From the deployer/admin perspective it is a
  significant easement, especially when lots of Service Providers are going to
  be configured (100s or more) - admin simply needs to upload auto-generated
  Metadata file, instead of making up URL-safe ``id`` for each ``Service
  Provider``. This would however change the way how ``Service Provider``
  objects would be identified.  This could be UUID hexadecimal values generated
  by ``Keystone`` instead of user specified ``id`` values.For a better
  ``Service Provider`` management HTTP GET call should be enhanced with filters
  where URL-safe ``entityID`` would be specified, for instance ``GET
  /v3/OS-FEDERATION/service_providers?entityID=http%3A//sp.example.com/Shibboleth.sso/ADFS``

Security Impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

It changes a Service Catalog but changes the structure in an additive fashion,
to not break existing consumers, not the set of data exposed.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

No.

* Does this change involve cryptography or hashing?

No.

* Does this change require the use of sudo or any elevated privileges?

No.

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

No.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

No.


Notifications Impact
--------------------

Please specify any changes to notifications. Be that an extra notification,
changes to an existing notification, or removing a notification.

No.

Other End User Impact
---------------------

python-keystoneclient would need to be enhanced with operations for
managing ``Service Provider`` objects, correctly interpret new structure of the
Service Catalog , list all the remote clouds/services a user can burst into
and reuse existing federated authentication plugins for the authentication
process.

Performance Impact
------------------

No performance impact.

Other Deployer Impact
---------------------

No additional config options, new features would be enabled only after the
``federation`` extension is enabled.


Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Marek Denis <marek-denis>
* Rodrigo Duarte <rodrigods>

Other contributors:
    None

Work Items
----------

* Deprecate ``url`` attribute in ``v3 regions``
* Add ``Service Provider`` objects along with relevant APIs
* Add ``service_providers`` object to the Service Catalog
* Document implemented changes

Dependencies
============

None.

Documentation Impact
====================

All the changes must be documented:
* New set of APIs
* New structure of the Service Catalog


References
==========

Etherpad site: https://etherpad.openstack.org/p/keystone2keystone

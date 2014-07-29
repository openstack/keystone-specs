..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
OpenID Connect federation
=========================

`bp openid-connect
<https://blueprints.launchpad.net/keystone/+spec/openid-connect>`_

As of the Icehouse release, the only federation protocol that is supported is
SAML, the purpose of this specification is to enable support for OpenID Connect
as a federation protocol.

Problem Description
===================

An Identity Provider (IdP) that issues and handles OpenID Connect requests
wishes to allow its users access to an OpenStack Cloud. Currently this is not
possible as the only federation protocol supported is SAML.

Proposed Change
===============

1. Create a new auth plugin or module for OpenID Connect requests.

2. Re-use the mapping engine to map any OpenID Connect attribues that are
   presented to Keystone attributes.

3. Leverage Apache plugin ``mod_auth_openidc`` to handle the requests.

Note: This feature should be written once the existing Federation code has been
re-engineered, as to avoid unnecessary code duplication.

Alternatives
------------

1. Add an authentication plugin that takes certain OpenID Connect arguments as
   parameters, and from the plugin we can send a request to the IdP, and
   handle the response within the plugin.

   The main issue with this alternative is that the user sends her credentials
   to the Service Provider instead of the Identity Provider.


Security Impact
---------------

Ensure we are following the OpenID Connect specifications.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

``python-keystoneclient`` would need to be enhanced to handle OpenID Connect
requests.

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

* stevemar (Steve Martinelli <stevemar@ca.ibm.com>)

* henrynash (Henry Nash <henryn@linux.vnet.ibm.com>)

Work Items
----------

* Add an OpenID Connect auth plugin to handle any necessary OpenID Connect
  specific data handling.

Dependencies
============

Relies on re-engineering federation for efficient implementation. See:
`<https://review.openstack.org/#/c/104301/>`_

Documentation Impact
====================

Extensive documentation will have to be provided to describe any new
configurations necessary.

References
==========

* `Blueprint
  <https://blueprints.launchpad.net/keystone/+spec/openid-connect>`_

* `Apache plugin mod_auth_openidc
  <https://github.com/pingidentity/mod_auth_openidc>`_

* `OpenID Connect Core Spec
  <http://openid.net/specs/openid-connect-core-1_0.html>`_

* `OpenID Connect Implicit Spec
  <http://openid.net/specs/openid-connect-implicit-1_0.html>`_

* `OpenID Connect Etherpad Details
  <https://etherpad.openstack.org/p/openidconnect>`_

* `Federation design session notes
  <https://etherpad.openstack.org/p/juno-keystone-federation>`_

* `Federation @ Atlanta Summit
  <http://dolphm.com/openstack-juno-design-summit-outcomes-for-keystone/#identityfederation>`_

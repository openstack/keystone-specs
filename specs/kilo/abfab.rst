..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
IETF ABFAB federation
=====================

`bp abfab <https://blueprints.launchpad.net/keystone/+spec/abfab>`_

As of the Icehouse release, the only federation protocol that is supported is
SAML, the purpose of this specification is to enable support for IETF ABFAB
as a federation protocol.

Problem Description
===================

An identity provider that issues and handles ABFAB requests wishes to
allow its users access to an OpenStack Cloud. Currently this is not possible as
the only federation protocol supported is SAML.

Proposed Change
===============

1. Create a new auth plugin or module for IETF ABFAB requests.

2. Re-use the mapping engine to map any IETF ABFAB attributes that are
   presented, into Keystone attributes.

3. Leverage the Moonshot implementation of ABFAB for Apache
   to handle the ABFAB protocol requests and responses.

4. ``python-keystoneclient`` would need to be enhanced to handle ABFAB
   requests.

Notes:

1. This feature should be written once the existing Federation code has been
   re-engineered, as to avoid unnecessary code duplication.
2. Patches are linked at the bottom of the spec.

Alternatives
------------

1. Add an authentication plugin to Keystone that directly handles the ABFAB
   protocol. From the plugin we can send an ABFAB request to the IdP through
   the encrypted EAP tunnel, then handle the response within the plugin.

   This will also work as ABFAB was designed to work in this mode, but it will
   mean that there is much more code that needs to be supported inside
   Keystone.

Security Impact
---------------

None, providing the Apache ABFAB plugin is implemented correctly and follows
the IETF specifications.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

Another ``python-keystoneclient`` spec should be made.

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

* d-w-chadwick (David Chadwick <d.w.chadwick@kent.ac.uk>)
* Ioram7 (Ioram Sette <iss@cin.ufpe.br>)

Work Items
----------

* Add an Apache ABFAB/Moonshot auth plugin to handle any necessary ABFAB
  specific data handling created by Apache.

Dependencies
============

None

Documentation Impact
====================

Extensive documentation will have to be provided to describe any new
configurations necessary.

References
==========

* `Blueprint
  <https://blueprints.launchpad.net/keystone/+spec/abfab>`_

* `Installing Moonshot on Apache
  <https://wiki.moonshot.ja.net/display/Moonshot/Apache+HTTPD>`_

* `RFC 7055
  <http://tools.ietf.org/html/rfc7055>`_

* `RFC 7056
  <http://tools.ietf.org/html/rfc7056>`_

* `RFC 7057
  <http://tools.ietf.org/html/rfc7057>`_

* `Federation design session notes
  <https://etherpad.openstack.org/p/juno-keystone-federation>`_

* `Federation @ Atlanta Summit
  <http://dolphm.com/openstack-juno-design-summit-outcomes-for-keystone/#identityfederation>`_

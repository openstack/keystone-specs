..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Native SAML in keystone
=======================

Blueprint `native-saml <https://blueprints.launchpad.net/keystone/+spec/native-saml>`_

We have had the ability for keystone to use a federated identity provider
since the Icehouse release. This is done using Apache modules like
`mod_shib`_ or `mod_auth_mellon`_. Unfortunately they have some key
limitations:

1. No REST API to manage IdPs or other configuration
2. Must restart/reload Apache to reload configuration

This works fine for small clouds where a single IdP is configured at bootstrap
time, but falls down in a larger cloud where those IdPs may be a little more
dynamic.

.. _mod_shib: https://wiki.shibboleth.net/confluence/display/SHIB2
.. _mod_auth_mellon: https://github.com/UNINETT/mod_auth_mellon


Problem Description
===================

Usecase:

1. As a public/private cloud operator I do not want to setup and manage the
   federation configuration for every domain. I would like to delegate that
   responsibility to the domain administrator. The domain admin can then make
   their domain members use whatever IdP they desire.


Proposed Change
===============

The following things would be changed in keystone:

1. Create APIs to manage IdPs in a backend
2. Create APIs to manage SAML attribute mapping in a backend
3. Create a new endpoint to do the SAML negotiation with an IdP

The code to interpret SAML2 would not be written by the keystone team. Instead
we will use the `pysaml2`_ library that we `already have as a dependency`_.

.. _pysaml2: https://github.com/rohe/pysaml2
.. _already have as a dependency: https://github.com/openstack/keystone/blob/470d92f/requirements.txt#L36

Alternatives
------------

There are a couple of alternatives:

1. Do nothing, but this doesn't make anyone happy.
2. Fix mod_shib/mod_auth_mellon to be more dynamic.

   It is certainly possible for a C expert to make the modules reload their
   configuration more dynamically. It's having APIs to manage the config that
   makes this much more difficult. keystone would need to write the XML config
   files and distribute then to all keystone nodes.

Security Impact
---------------

This code will be managing the negotiation between keystone and the IdP. There
is certainly an impact if we get it wrong. In the worst case we are opening a
new attack vector.

We are also making use of the pysaml2 library to do the heavy lifting. There
is extra onus on us to make sure that code is correct and have a good process
for dealing with security issues.

We will be using x509 certificates to validate and sign SAML documents. Those
certificates will need to be managed and rotated by the operator. (This is no
different for operators that are currently using an Apache module for
federation.)

Notifications Impact
--------------------

We may possibly have notifications for some IdP interactions. Nothing concrete
yet.

Other End User Impact
---------------------

End users should not be able to tell that we are providing the federation bits.
It will all be transparent to them.

Performance Impact
------------------

This should have no direct significant performance impact. Signing/verifying
documents may block a process a little longer, but that shouldn't impact
keystone all that much.

Other Deployer Impact
---------------------

1. Certificate rotations documented in the `Security Impact`_ section.
2. Middleware will have to be utilized to take advantage of the feature.
3. Deployment processes will have to use APIs to manage IdP configuration
   instead of managing the existing XML files.


Developer Impact
----------------

Nothing significant.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek (David Stanek <dstanek@dstanek.com>)

Work Items
----------

Work items are documented in `Proposed Change`_


Dependencies
============

Just the `pysaml2`_ library.


Documentation Impact
====================

1. We'll need to change/augment federation documentation to show how to setup
   and use this feature.
2. Updates will need to be made to the keystone API documentation


References
==========

None

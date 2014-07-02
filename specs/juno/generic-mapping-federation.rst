..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Standardizing the federation process
====================================

`bp generic-mapping-federation
<https://blueprints.launchpad.net/keystone/+spec/generic-mapping-federation>`_


We propose that the federated authentication process be reengineered so that a
user_id is returned alongside the identity attributes. In addition, the saml2.py
authentication module should be renamed as mapped.py to denote that it applies
to all authentication methods which require the asserted identity attributes to
be mapped into local attributes.


Problem Description
===================

The existing federation logic uses `saml2` auth plugin to establish a local
Keystone identity, by mapping trusted external identity attributes to local
identity attributes. Now that other federation protocols will be supported by
Keystone, it makes no sense to apply mapping via a saml2 plugin. Some federation
authentication protocols already determine the ID of the user, this should be
leveraged by the protocol specific sub modules and not the mapping engine.

Proposed Change
===============

In order to resolve the problem the following changes should be made:

#. The saml2.py file should be renamed as mapped.py.
#. The method name "saml2" should be removed from mapped.py.
#. The renamed plugin Saml2 -> Mapped should have the logic for attribute
   mapping and attribute extraction split.
#. The function for extracting attributes should return a user_id among the
   asserted attributes.

Splitting the logic of the Mapped plugin will allow subclassing of the plugin to
handle protocols which provide attributes in a different way to mod_shib, while
still enabling reuse of the mapping filter implementation. We can then use the
correct method name according to the protocol used so that clients can easily
determine their behaviour.

For instance, a configuration for a SAML2 and an Open ID Connect enabled
Keystone might look like: ::

  [auth]
  methods = username, password, saml2, openidc
  saml2 = keystone.auth.plugins.mapped.Mapped
  openidc = keystone.auth.plugins.mapped.Mapped

This assumes that both protocols manage attributes in the same way so that
extraction is identical. If this is not the case, the configuration might look
like: ::

  [auth]
  methods = username, password, saml2, openidc
  saml2 = keystone.auth.plugins.mapped.Mapped
  openidc = keystone.auth.plugins.mapped.OpenIDCMapped

Where OpenIDCMapped is a subclass of Mapped which handles its specific
extraction of assertion data, then calls the mapping layer handling of the
Mapped class.


Alternatives
------------

Alternatively no changes could be made but this is undesirable as it makes no
sense for non SAML assertion data to be processed by a module named saml2.py.

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

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

This would force developers to implement new federated protocols in a
standardized way by requiring the attribute extraction / retrieval process to
return a user_id among the asserted attributes.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Kristy Siu kwss@kent.ac.uk IRC: kwss

Other contributors:
  David Chadwick d.w.chadwick@kent.ac.uk

Work Items
----------

* Reengineer the current federation process to ensure that a user_id is
  guaranteed to be returned with the assertion attributes.
* Modify handling of assertion data extraction to work across multiple
  protocols.
* Rename saml2.py to mapped.py and remove the method name "saml2" from this.
* Split the mapping and attribute assertion logic to allow protocol specific
  subclassing.

Dependencies
============

None

Testing
=======

Current federation tests should still provide coverage.

Documentation Impact
====================

Development docs should specify that functions to extract asserted attributes
received from external identity providers should always return a user_id as
part of the assertion data.

References
==========

None

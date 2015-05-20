..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Domain Configuration Storage
============================

`bp domain-config-default <https://blueprints.launchpad.net/keystone/+spec/domain-config-default>`_


Provide the ability for the default values for domain-configurable options to
be retrieved via the API.


Problem Description
===================

Domain specific option configuration via the API was introduced in Kilo:
https://github.com/openstack/keystone-specs/blob/master/specs/kilo/domain-config-ext.rst

This provides the crud mechanism to manager domain specific options.  However,
it does not allow a domain admin to discover the default settings for any such
options - and hence the knowledge of whether they need to override a specific
option for a given domain.


Proposed Change
===============

This proposal will extend the domain config API to allow the retrieval of the
default options for those that can be configured on a domain basis, allowing
a domain administrator to then decide whether to use the existing facilities
of the domain config API to override any of these values.

It does not provide an ability to set the global default values for a keystone
server - these can still only be changed via the main keystone configuration
file.

Alternatives
------------

Rather than provide a specific API to retrieve the defaults, we could modify
the current API (which is marked as experimental) to simply return the current
value for any option, whether it is the default or whether it has been set
explicitly by the API for this domain. While this is perhaps simpler for those
just reading the option values, it would complicate the semantics of the APIs
to modify such options. For instance, you would somehow need to represent in
the API the fact that if you deleted an option that was set for a domain, and
then read it back, you would still get a value...it would just be the default
value.

Data Model Impact
-----------------

None

REST API Impact
---------------

The exact API specification will be defined as part of a review of
changes to the Identity API.

Security Impact
---------------

This functionality exposes a new API to the backend configuration data for a
domain. Like any other v3 API, it will be subject to the standard RBAC
permissions model.

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
    henry-nash

Work Items
----------

- Get agreement of API specifications
- Implement the domain configuration code
- Add support to keystoneclient library
- Add support to openstack client

The work for supporting this API in Horizon will be proposed separately.

Dependencies
============

None

Testing
=======

None, above and beyond unit testing

Documentation Impact
====================

Changes to the Identity API and configuration.rst.

References
==========

None

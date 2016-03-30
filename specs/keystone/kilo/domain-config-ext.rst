..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Domain Configuration Storage
============================

`bp domain-config-ext <https://blueprints.launchpad.net/keystone/+spec/domain-config-ext>`_


Provide the means to store the configuration options that would normally stored
in domain-specific configuration files in an SQL table instead.


Problem Description
===================

Domain specific configuration files are used to specify the unique options
for a given domain with regard to its identity backend (for example
``[ldap] url`` to be used for that domain). This use of a separate
configuration file for each domain leads to some complexity in the
choreography of on-boarding a new domain, e.g.:

* Call the Keystone API to add a new domain, create whatever domain role
  assignments are required
* Use an out-of-band mechanism to create and place the domain specific
  configuration file on the Keystone server
* Restart Keystone to ensure cause the domain specific configuration file to
  be read

The two different mediums of where information about a domain is stored (i.e.
identity driver information in the configuration file as well as domain name,
description etc. in SQL) also makes it rather difficult to easily see what
options are defined for a specific domain.

Proposed Change
===============

This proposal will provide the ability to store the configuration options in
an SQL backend, indexed by domain ID. A main configuration option will dictate
whether Keystone uses this extension to obtain domain specific
configuration data or the existing domain specific configuration files
approach. It will be an all or nothing option - there will be no mixing of some
options in the extension and some in configuration files. If the extension is
specified as the store of configuration data, then any domain-specific
configuration files will be ignored. Keystone will go through the logic of
building the domain configurations in the same way as it does today, either
reading the configurations options from the extension or from the set files.

Whenever any of the domain-specific configuration options are updated, the
identity backend for a given domain will be re-loaded (using whatever new
configuration data there may be), further assisting the on-boarding of a
domain.

This new capability can only be used to replace options that would have
appeared in a domain specific file - it will not allow options to be specified
that would normally only appear in the main Keystone configuration file. Since
the options are available via REST, the openstack client and Horizon can
provide the ability to view and set this information.

Although the configuration options will be stored in SQL, this does
not imply anything about the medium of the identity backends themselves (which
could be all LDAP if required).

One concern over this ability would be that some configuration options may
be considered highly sensitive and would want to be stored separately and
while can be written via the API, they cannot be read (for example password
of the user required for doing LDAP searches, in the case when anonymous query
is not supported). To address this concern, it is proposed that internally we
enforce an explicit whitelist of config options that can be read. All others
will be stored in a separate backend table and not returned on read. The
current proposal is that all config options except password will be included
in the whitelist. This means that the LDAP url will be returned on read (since
that will be a common option cloud administrators might want to check). There
may be situations where the url is required to encode a non-whitelisted item
(e.g. password). To avoid these then being visible on read, we will support the
substitution commands within any config strings that allow any non-whitelisted
items to be referenced, by specifying within the option:

%(config-option-name)s

For example:

"url": "http://myldap/root/henry/%(password)s"

The configuration option specified for substitution, which must exist in the
same group as the option being defined (in this case "url"), will be
substituted when Keystone uses the configuration option internally.

This new capability will initially be classed as experimental in-tree
functionality, with the intention of migrating to stable as soon as possible,
with a stretch goal of this occurring before the release of Kilo.

Alternatives
------------

Use of federation to access IdPs would (effectively) move this problem to
creating mapping rules.

Data Model Impact
-----------------

Changes to the data model will be restricted to a new tables for storing the
configuration information.

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

None, other than this functionality will subscribe to domain deletion events.

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

For cloud providers who have already deployed a domain-specific installation,
a one-shot option to keystone-manage will be provided that will copy the
contents of the domain-specific configuration files into the SQL store.

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
- Implement the keystone-manage migration option
- Add support to keystoneclient library
- Add support to openstack client

The work for supporting this API in Horizon will be proposed separately.

Dependencies
============

None

Testing
=======

Beyond the regular unit testing, there will be testing for the migration
options.

Documentation Impact
====================

Changes to the Identity API and configuration.rst.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Multi-Attribute Endpoint Grouping
=================================

Out of the box, when Keystone issues a project scoped token, it returns all
known service endpoints regardless of whether users have access to them or not.
This is neither efficient nor secure. In the Havana release, an endpoint filter
extension was created. Unfortunately this offers limited filtering capabilities
and furthermore is difficult to manage service endpoints. The purpose of this
specification is to outline an improved endpoint filtering solution.

`bp multi-attribute-endpoint-grouping
<https://blueprints.launchpad.net/keystone/+spec/
multi-attribute-endpoint-grouping>`_

Problem Description
===================

At present with the Endpoint Filtering extension in Keystone, we are able to
associate endpoints with projects, thereby limiting the service endpoints that
appear in a catalog for a project scoped token. However, anyone who belongs to
a given project can see all the project scoped service endpoints. From a
Security perspective this clearly violates the principle of 'Least Privilege'.
For instance, a Glance Administrator on the US West Coast should not see Glance
or for that matter any other service endpoints on the US East Coast or any
other region for that matter. Therefore there is a strong need to be able to
offer more fine grained capabilities for filtering service endpoints returned
in the service catalog. In particular, we should be able to filter service
endpoints according to service (e.g. Nova, Swift, etc) or interface type
(e.g. admin, internal, etc) or region or any combination thereof.

In addition, with the present Endpoint Filtering extension, managing service
endpoints is a challenge due to the static nature of the relationship between a
service endpoint and a project. For example, currently you must explicitly
create a project association for every new service endpoint which is a
management headache. Essentially the current solution provides a static group
of endpoints associated with a given project.

Proposed Change
===============

Multi-Attribute Endpoint Grouping essentially builds upon and extends the
capabilities offered by the Endpoint Filtering extension by introducing a
dynamic endpoint attribute filtering capability that is directly associated
with a project.

The underlying idea of Multi-Attribute Endpoint Grouping is to provide a
key-value based filtering strategy that groups service endpoints having the
same characteristics (e.g. service, type, region).

The benefits of Multi-Attribute Endpoint Grouping are as follows:

* Service endpoints can be easily managed according to their characteristics
  which can act as filters. A list of filters would be: ``service_id``,
  ``service_type``, ``region_id``.

* Service endpoints can belong to multiple groups which increases the level of
  granularity. For instance, you can limit service endpoints to a certain
  service (e.g. Swift) and region (e.g. US-West).

* Multi-Attribute Endpoint Grouping provides a superset of the Endpoint Filter
  functionality that was added to Keystone as an extension in Havana.

Alternatives
------------

An alternative solution to Multi-Attribute Endpoint Grouping is to be able to
define fine-grained rules in the policy file that dictate which service
endpoints be included in the catalog for a given project scoped token. At the
moment, such fine-grained dynamic rules are not feasible, in part, because both
the policy language and Keystone's interpretation are insufficient but also due
to the static nature of policy configuration, limits dynamically adding new
service endpoints at runtime.

Security Impact
---------------

This extension is not intended to provide additional security. Instead,
endpoints are simply omitted from the user's service catalog. This behavior
does not prevent users from communicating with an endpoint.

The benefit of this proposal is that we offer more fine grained control over
what service endpoints are returned to the user in the underlying service
catalog.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Generally speaking this change should reduce the size of the service catalog
included in project scoped tokens since we are filtering service endpoints.
Each project scoped token request will require an additional SQL query to find
all endpoint groups associated with the target project. And then one SQL query
per associated endpoint group. This additional querying should add minimal
performance impact and by default filtering is not enabled.

Other Deployer Impact
---------------------

Deployers will need to enable the endpoint filter extension and create the
endpoint filter extension tables. This information will be documented in
``keystone\doc\source\extensions``.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Bob Thyne (bobt)

Other contributors:
  Fabio Giannetti (fabiog)

Work Items
----------

* Update the existing endpoint filter as follows:
    * Add SQL for creating the new multi-attribute endpoint grouping tables.
    * Add SQL for CRUD operations associated with new multi-attribute endpoint
      grouping operations.
    * Add SQL for dealing with migration to the new multi-attribute endpoint
      grouping.
    * Update the router, controller and core to deal with the new APIs
      associated with the multi-attribute endpoint filter.
    * Update/Expand the unit tests to cover the new multi-attribute endpoint
      grouping APIs.
    * Update the policy.json to ensure admin privilege is required for the new
      multi-attribute endpoint grouping APIs.
* Create Documentation including setup and configuration

Dependencies
============

None

Documentation Impact
====================

* The Multi-Attribute Endpoint Grouping extension API will need to be
  documented (api changes).

References
==========

* https://blueprints.launchpad.net/keystone/+spec/endpoint-filtering

* https://blueprints.launchpad.net/keystone/+spec/multi-attribute-endpoint-grouping

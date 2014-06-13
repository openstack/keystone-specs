..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Endpoint Policy Extension
=========================

`bp endpoint-policy <https://blueprints.launchpad.net/keystone/+spec/endpoint-policy>`_


Extension ``OS-ENDPOINT-POLICY`` provides the means to assign policy files to
specific endpoints. This extension requires v3.3 of the Identity API.


Problem Description
===================

OpenStack provides RBAC capabilities for controlling access to project-based
resources.  It does not, however, provide RBAC capabilities to control which
endpoints can consume such resources.

The classic use case that illustrates this problem is a cloud that is divided
into collections of endpoints that represent, say, Test, QA and Production
areas. In such environments, you would typically want to restrict the users who
can issue API calls that affect production, from those that can access the Test
environment for example. Today, the only way of achieving this is to have a
different project for each environment and use endpoint filtering to restrict
the set of endpoints to each one (and then have different role requirements for
each of those projects). This can rapidly lead to project-explosion, especially
when you consider you might have different geographical production regions,
perhaps each with their own admin users. What is really required is to vary the
RBAC requirements by endpoint.

Proposed Change
===============

This change provides the ability to associate a given policy file (stored in
Keystone and represented by a policy ID) with an endpoint or set of endpoints.
Three types of association will be supported:

- A policy associated with a specific endpoint
- A policy associated with a service in a given region
- A policy associated with a service

The existing Identity policy API will be extended to allow a policy to be
retrieved for a specific endpoint, where the association created for that
endpoint will be used to select the appropriate policy file, by checking
the above three types of association *in that order*. For regions that
have parents, these will be searched in ascending order. The first
association match found will be used - no combining of policy files due to
multiple associations will be carried out. If no association is found, then an
error will be returned.

This extension will not change the existing API allowing the retrieval of
policy by policy ID, rather it will provide an alternative where an endpoint
has the ability to retrieve an appropriate policy rather than having to specify
it by policy ID. Given that, in this current proposal, it as assumed that an
endpoint will continue to only fetch its policy at startup. Future
enhancements, not covered by this proposal, might allow for Keystone to
provide notifications upon policy changes to allow endpoints to refresh
their policy.

This change would allow a cloud administrator to, for example, specify a
restrict set of role assignments that would be needed to undertake certain
actions in a production region.  For instance, a cloud provider might define
a nova policy that, for production, has rules like:

.. code-block:: javascript

    "compute_extension:admin_actions:pause": "rule:admin_or_owner",

While in a test region, the policy rules may be more relaxed, for example:

.. code-block:: javascript

    "compute_extension:admin_actions:pause": "rule:admin_or_owner or role:tester",

By associating the two policy files with the nova service for their respective
regions, the correct policy file would automatically by returned to any nova
endpoint in either region if it fetched its policy by endpoint ID.

Alternatives
------------

The current solution (of having multiple projects), in conjunction with
endpoint filtering could continue to be used, with the resulting problems
described above.


Data Model Impact
-----------------

Changes to the data model will be restricted to new tables for storing the
endpoint policy associations.

REST API Impact
---------------

The exact API specification will be defined as part of a review of an
changes to the Identity API.  This will include:

- API to define policy associations
- Extension to the existing Identity policy API to ask for policy

Security Impact
---------------

Above and beyond the act of allowing endpoints to get policy from Keystone
(which is already supported), there is no significant security impact.

Notifications Impact
--------------------

All policy association changes will be auditable events.

Other End User Impact
---------------------

Endpoints will need to call Keystone to obtain their policy (rather than
read it from a file) in order to take advantage of this change. Keystone
middleware should provide support for this.

Performance Impact
------------------

Negligible - policy will be fetched infrequently.

Other Deployer Impact
---------------------

Endpoints will need a configuration setting to tell them for where to obtain
their policies (i.e. file or Keystone).

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
    henry-nash

Additional assignee:
    ayoung

Work Items
----------

- Get agreement of API specifications
- Implement the policy association code
- Extend the policy API to enable the use of any associations
- Modify Keystone middleware to expose this new capability

Separate patches/blueprints will be used to modify the policy loading of other
projects to be able to use this capability.

Dependencies
============

Changes to the policy loading of other projects will be needed before the
advantages of this extension can be obtained.

Testing
=======

Due to the interdependent nature of the changes of this proposal, some Tempest
tests will be provided.

Documentation Impact
====================

Changes to the Identity API and configuringservices.rst.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Add Endpoint Filter Enforcement to Keystonemiddleware
=====================================================

Problem Description
===================

In Keystone, we have the ability to filter endpoints in the service catalog.
However, at run-time we do not enforce that a target service endpoint actually
exists in the service catalog. This means that a user with a valid token can
access any service endpoint. Of course, additional security layers such as
roles based access control will limit the damage.

Nevertheless, in a holistic security environment, offering the ability to
provide layered security such as endpoint enforcement is important. Especially
in the case of global roles whereby an adminstrator of one service by default
has administrator access to all services in a vanilla Openstack installation.

Proposed Change
===============

The proposed solution is to create an optional endpoint enforcement check
within auth_token that will verify a specific endpoint is contained in the
service catalog associated with a given token. If the target endpoint is not
contained in the service catalog then the request will be rejected. The target
service endpoint will be configurable as a middleware parameter together with
an option to enable or disable endpoint filtering which will be disabled by
default.

The `auth_token` middleware will have 3 new options. To enable hard enforcement
the configuration will need to include the endpoint's `service_id`, the
`endpoint_id`, and the boolean `enforce_endpoint_in_authz`. If either
`service_id` or `endpoint_id` are missing the `enforce_endpoint_in_authz`
will be implicity false.

Alternatives
------------

Two alternatives:

* An existing Keystone spec called ``Token Constraints`` talks about adding
  endpoint enforcement via token constraints. Our proposal focuses on endpoint
  enforcement via the service catalog. The advantage with this approach is
  that the change is small and restricted to the Keystone middleware layer.

* An alternative implementation is to create a new middleware component that
  is positioned after auth_token in the pipeline which will perform the
  endpoint enforcement logic. The disadvantage with this implementation
  approach is that we create a new middleware component that needs configured
  and we duplicate logic that exists in auth_token.

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

None - endpoint enforcement will be turned off by default. If enabled then
we search the service catalog for a given endpoint which will have a neglible
impact on performance.

Other Deployer Impact
---------------------

If enabled, deployers need to configure auth_token to turn on endpoint
enforcement and define the target service endpoint to be matched. We assume
the Service Catalog is filtered.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bobt

Other contributors:
  None

Work Items
----------

* Add endpoint enforcement logic in auth_token

* Update ``keystonemiddleware`` with new enforcement configuration options

* Add enforcement logic to ``keystonemiddleware`` consuming the config options

Dependencies
============

None

Documentation Impact
====================

Update ``keystonemiddleware`` docs to include how to enable and configure
endpoint enforcement.

References
==========

* `Token Constraints Spec
  <https://review.openstack.org/#/c/123726>`_

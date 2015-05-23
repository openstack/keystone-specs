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
access any service endpoint.

Of course, additional security layers such as roles based access control will
limit the scope of this insecurity but nevertheless, in a holistic security
environment, offering the ability to provide layered security such as endpoint
enforcement is important.  This is particularly true in the case of global
roles such as an administrator of one service in a vanilla OpenStack
installation who by default will have administrator access to all services.

Proposed Change
===============

The proposed solution is to add the endpoint constraint enforcement capability
to the existing `auth_token` middleware. Endpoint constraint enforcement will
be based on a given global rule in the service's (Oslo) policy file
matching the endpoint IDs passed in the token. The given rule, if
exists, will be matched against the endpoints found request token's
service catalog. If there's at least one match, user is allowed to access the
endpoint. Otherwise, an endpoint access denied exception will be thrown. Since
endpoint constraint enforcement is part of token validation logic, an endpoint
access denied exception is the same as `InvalidToken` exception. Therefore, the
existing logic for handling `InvalidToken` exception remains unchanged. For
example, if the `delay_auth_decision` is set to True, request will still be
propagated down the pipeline despite the endpoint validation failure.

The `auth_token` middleware will have two new options.

    `enforce_global_target`	- enable global rule enforcement. Default is
                                  False.
    `global_target_name`	- name of the global target in the policy file
                                  to enforce. Default is `global`.


For example, your policy file should contain something like this::

    {
        ...
        "endpoint_binding": "token.catalog.endpoints.id=%{CONF.endpoint_id}s",
        "global": "rule:endpoint_binding",
        ...
    }

Policy configuration comes from the service's global configuration file.
For example::

    [oslo_policy]
    policy_file = policy.json

If `enforce_global_target` is set to False, endpoint constraint will not
be enforced.

If `enforce_global_target` is set to True and global target is not found in
service's policy file, a `ConfigurationError` exception will be raised.

If `endpoint_global_target` is enabled and service catalog is not found in
token data, middleware will attempt to fetch to service catalog from Keystone
before performing the enforcement.


Alternatives
------------

An existing Keystone spec called ``Token Constraints`` talks about adding
endpoint enforcement via token constraints. Our proposal focuses on endpoint
enforcement via the service catalog. The advantage with our approach is
that the change is small and restricted to the Keystone middleware layer.


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

None - global target enforcement will be turned off by default. If enabled then
the service catalog will be processed to establish compliance with the
configuration.  No additional calls to keystone will be necessary so
Impact on performance will be neglible.

Other Deployer Impact
---------------------

To enable and activate the global target enforcement the deployer must define
a new rule in their policy.json with a target name that matches that
configured for `global_target_name`.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  kennedda (David C Kennedy)

Other contributors:
  gyee (Guang Yee)

Work Items
----------

* Add the global target enforcement capability to `auth_token` filter

* Update ``keystonemiddleware`` with new enforcement configuration options

* Add enforcement logic to `auth_token` filter consuming the config
  options

Dependencies
============

None

Documentation Impact
====================

Update ``keystonemiddleware`` docs to include how to enable and configure
endpoint enforcement via global target.

References
==========

* `Token Constraints Spec
  <https://review.openstack.org/#/c/123726>`_

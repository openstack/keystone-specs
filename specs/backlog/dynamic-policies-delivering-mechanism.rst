..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
 Centralized Policies Distribution Mechanism
=============================================

`bp dynamic-policies-delivery <https://blueprints.launchpad.net/keystone/+spec/dynamic-policies-delivery>`_

This spec defines the Identity server side implementation of the Centralized
Policy distribution as defined by the Keystone Middleware spec Centralized
Policies Fetch and Cache [#middleware-spec]_. Please refer to it for further
background, motivation and use case information.

The Identity server will control the cache in endpoints through appropriate
HTTP cache control headers, in order to keep policy consistent across different
service endpoints which must have the same Centralized Policy, for example,
multiple Nova processes running behind a proxy.


Problem Description
===================

Please refer to the Keystone Middleware spec [#middleware-spec]_.


Proposed Change
===============

The proposed change is to add to the Identity server the capability to control
policy cache in remote service endpoints by using the appropriate HTTP cache
control headers.

Freshness
---------

This defines how long the retrieved policy is fresh for. This time delta will
be defined in the ``Cache-Control`` HTTP header included in both
``GET /endpoints/{endpoint_id}/OS-ENDPOINT-POLICY/policy`` and
``GET /policies/{policy_id}`` responses. It will be respected by the Keystone
Middleware implementation running at the service endpoints side.

In order to control the freshness for policy entities, the Identity server will
have a config option called ``policy_cache_time`` to define the timeout.

If we call 'checkpoint' the event of timeout expiration, the freshness for a
given policy is calculated based on the current time and when the next
checkpoint will occur. Take the following situation as example: ::

    The Identity server has ``policy_cache_time`` set to 300, i.e there is a
    checkpoint every 300 seconds.

    If there is a policy request 140 seconds after the last checkpoint, the
    Identity server will return the policy and say it is only fresh for 160
    seconds, when the next checkpoint will occur at the server side.

Policy Consistency
------------------

In order to keep policies consistent between endpoints which share the same
Centralized Policy, the Identity server will create versions of policies, which
are released every checkpoint.

When a policy request hits the Identity server, it will deliver the last
released policy, ensuring that different endpoints asking the server at
different times inside the ``policy_cache_time`` slice will get the same
policy.

As a side effect, policy updates happening between two checkpoints will only
take effect in the next release of that policy.

Thundering Herd
---------------

This problem can occur if ``policy_cache_time`` times out at the same time for
all the policies in the system at the Identity server, then all the service
endpoints would be asking for policy updates at the same time.

The proposed approach is to spread sets of endpoints which share the same
policy over the time. This can be done by making the freshness of policies
expires according a delay defined in fuction of their IDs, for example: ::

    delay(policy-id) = hash(policy-id) % ``policy_cache_time``

Further HTTP Headers
--------------------

Besides the cache control HTTP headers, the following will be specified:

* ``must-revalidate``: if the response is stale, all caches must first
  revalidate it with the server;
* ``private``: indicates that the response message is intended for a single
  endpoint and must not be cached by a shared cache.

Alternatives
------------

Please refer to the Keystone Middleware spec [#middleware-spec]_.

Security Impact
---------------

Please refer to the Keystone Middleware spec [#middleware-spec]_.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Please refer to the Keystone Middleware spec [#middleware-spec]_.

Other Deployer Impact
---------------------

No impact on server side, however some configuration impact is expected in the
service endpoints side, in the Keystone Middleware configuration, as specified
in the Keystone Middleware spec [#middleware-spec]_.

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Samuel de Medeiros Queiroz - samueldmq

Other contributors:

* Adam Young - ayoung
* You ?

Work Items
----------

* Add the capability of policy releases management;
* Introduce the ``policy_cache_time`` option and manage timeouts for different
  policy entities;
* Include appropriate HTTP headers in the response of GET policy entities.


Dependencies
============

A list of related specs defining the dynamic delivery of policies can be found
under the topic `dynamic-policies-delivery <https://review.openstack.org/#/q/topic:bp/dynamic-policies-delivery,n,z>`_.


Documentation Impact
====================

None besides the regular API documentation.


References
==========

.. [#middleware-spec] Keystone Middleware Spec: `Centralized Policies Fetch and Cache <https://review.openstack.org/#/c/134655/>`_.


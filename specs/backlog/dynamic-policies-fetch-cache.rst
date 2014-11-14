..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
 Centralized Policies Fetch and Cache
======================================

`bp centralized-policy-delivery <https://blueprints.launchpad.net/keystone/+spec/centralized-policy-delivery>`_

OpenStack uses a Role-Based Access Control mechanism to manage authorization,
which defines if a user is able to perform actions on resources based on the
roles he has assigned on them. Resources include VMs, volumes, networks, etc
and are organized into projects, which are owned by domains. Users have roles
assigned on domains or projects.

Users get domain or project scoped tokens, which contain the roles the user has
assigned on them, and pass this token along to services in requests to perform
actions on resources. The services check the roles and the scope from the token
against the rules defined for the requested action on the policy.json file to
determine if the user's token has enough privileges to execute it.

In order to manage access control to services in an OpenStack cloud, operators
need to use an out-of-band mechanism to update and distribute the policy.json
files to the appropriate service endpoints.

`Dynamic Policies <https://wiki.openstack.org/wiki/DynamicPolicies>`_ aim to
improve access control in OpenStack by providing an API-based mechanism for
defining and delivering policies to service endpoints.

In terms of definition, policy rules will be managed in a centralized approach
in the Identity server. They would be called Centralized Policies and can be
associated to service endpoints using the endpoint policy API.

What this spec proposes is that, once the Centralized Policies are defined and
associated, they will be fetched and cached by the associated service endpoints
by Keystone Middleware.


Problem Description
===================

Adjusting access control rules to better match the organization's policy is a
task done by deployers without the support of any OpenStack API.

Today, deployers update their policy files locally and use Configuration
Managerment System (CMS) tools to distribute them to the appropriate service
endpoints.

This approach presents a limitation that could be mitigated if the policy
definition and distribution were done via API, that is the process of keeping
the CMS tools in sync becomes a laborious task when the topology changes
frenquently, for example when there is a variable number of service nodes
behind proxies.


Proposed Change
===============

The Identity server already allows operators to define policy rules and manage
them via the `Policy API <http://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3.html#policies>`_.

Those Centralized Policies can be associated to service endpoints through the
`Endpoint Policy API <https://github.com/openstack/keystone-specs/blob/master/api/v3/identity-api-v3-os-endpoint-policy.rst>`_,
which allows policy entities to be associated directly to service endpoints,
services or regions.

The proposed change is the distribution of those policies to service endpoints
transparently, by adding to Keystone Middleware the capability to fetch and
cache them for the endpoint it is serving.

This mechanism will be controlled by the Identity server, as described in the
spec `Centralized Policies Distribution Mechanism <https://review.openstack.org/#/c/197980/>`_.

The policy fetch will be based on the ``endpoint_id`` config option. This would
ease the task of keeping an external CMS tool in sync with the topology of the
endpoints in the cloud, since multiple processes behind a proxy would have the
same config.

Once ``endpoint_id`` config is known, Middleware requests the Identity server
for the respective Centralized Policy associated with it.

After fetching the policy rules, they will be cached according to the
appropriate HTTP header values from server responses.

Thus, take the following response as example: ::

    HTTP/1.1 200 OK
    Cache-Control: max-age=300, must-revalidate, private
    Last-Modified: Tue, 30 June 2015 13:00:00 GMT
    Content-Length: 1040
    Content-Type: application/json

    { ... }

Freshness
---------

This defines how long the retrieved policy is fresh for. This time delta is
defined by respecting ``Cache-Control`` HTTP header from server response. Once
the freshness is over, the Identity server will be asked again for an update.

Alternatives
------------

1. Leave the Configuration Management System (such as Puppet) manage it;

This would keep the out-of-band mechanisms used today to update and distribute
the policy files to endpoints.

2. Add a policy management API on each OpenStack project

This alternative is opposed to the centralized policy management. The new API
would be reflected on every service endpoint. There are many drawbacks in this
proposal, such as:

* If one wants to update the policy for all Nova endpoints, how to know them
  all ? How would it cope with proxies ?
* Defining an API that must be consistently implemented through all the
  OpenStack services is a very slow process and would have the same final
  effect as centralizing it in the Identity server.

Security Impact
---------------

This change touches policy rules, which are sensitive data since they define
access control to service APIs in OpenStack.

A potential attack vector is that a user who acquires access to the policies
API in the Identity server would be able to change the centralized policy
definition and association to service endpoints, being able to change access
control of any service endpoint using this feature in the cloud.

Documentation exposing this security risk will be provided to warn deployers
that access to the policies API must be very restricted.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Performance will be impacted when the cache has expired and Keystone Middleware
needs to ask the Identity server to update it. If the Identity server has an
update for it, the processing done by Middleware would need to ask
``oslo.policy`` to update the Centralized Policy, which requires I/O
operations.

At enforcement time, ``oslo.policy`` will also need to load the Centralized
Policy file to consider its custom rules when doing enforcement. Performance
may be slightly impacted at this point as well.

Benchmarking tests will be performed in a topology where there are multiple
processes running behind an HAProxy. The results will be posted in the Keystone
performance `wiki page <https://wiki.openstack.org/wiki/KeystonePerformance>`_.

Other Deployer Impact
---------------------

A config switch called ``enable_centralized_policy`` will allow deployers to
easily enable and disable the fetch and cache of Centralized Policies. It
defaults to ``false``, meaning that the old policy mechanism will be used by
default, since no policy will be fetched from the server.

In addition, deployers may need to define the ``endpoint_id`` config for each
service endpoint, as they have their own middleware filter defined in their
WSGI pipeline.

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

Work Items
----------

* Introduce ``enable_centralized_policy`` and ``endpoint_id`` config options;
* Add the capability to identify when the cache has expired and then fetch an
  update from the Identity server and call ``oslo.policy`` to overlay the
  existing local policy file.


Dependencies
============

A list of related specs defining the dynamic delivery of policies can be found
under the topic `dynamic-policies-delivery <https://review.openstack.org/#/q/topic:bp/dynamic-policies-delivery,n,z>`_.


Documentation Impact
====================

Documentation will be provided with the Keystone Middleware config options.


References
==========

* `Policy API <http://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3.html#policies>`_;
* `Endpoint Policy API <https://github.com/openstack/keystone-specs/blob/master/api/v3/identity-api-v3-os-endpoint-policy.rst>`_.

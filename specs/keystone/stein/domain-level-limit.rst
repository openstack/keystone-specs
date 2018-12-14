..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Domain Level Unified Limit Support
==================================

`bp domain-level-limit <https://blueprints.launchpad.net/keystone/+spec/domain-level-limit>`_

Currently Unified limit in Keystone is for project level. It can't cover the
domain level case.

Problem Description
===================

In some case, especially for Public Cloud, an account in the cloud is usually
a domain in Keystone. In the domain(account), the cloud provider allow the user
to create his own projects, users, or roles. So of course, these resource
should be handled by limit as well. From service provider sight, he needs the
ability to control the cloud account's limit. But Keystone now doesn't support
domain level limit. Suppose a case: There is a domain(account) has 3 projects,
the service provider want to limit each project can create 10 vms, but the
total vms in the domain should be less than 20. Keystone can't handle this case
currently.


Proposed Change
===============

Database change
---------------

Create a new column called `domain_id` to limit table to store the limit's
`domain_id` proprety.

API change
----------

Since the `domain` related APIs and resources are still alive in Keystone, API
callers usually treat `domain` different with `project`. So for unified limit
APIs, we'll add `domain_id` as well.

**Request:** ``POST /limits``

Add `domain_id` into request body. Either `project_id` or `domain_id` must be
provided. The body is like:

**Request Body**

.. code-block:: json

    {
        "limits":[
            {
                "service_id": "9408080f1970482aa0e38bc2d4ea34b7",
                "project_id": "3a705b9f56bb439381b43c4fe59dccce",
                "region_id": "RegionOne",
                "resource_name": "snapshot",
                "resource_limit": 5
            },
            {
                "service_id": "9408080f1970482aa0e38bc2d4ea34b7",
                "domain_id": "4d705b9f56bb296381b43c4fe59da34d",
                "resource_name": "volume",
                "resource_limit": 10,
                "description": "Number of volumes for domain 4d705b9f56bb296381b43c4fe59da34d"
            }
        ]
    }

**Request:** ``GET /limits/{limit_id}``

The response body will contain `domain_id` as well. If the `project_id` in the
response body is None, it means this is a domain level limit.

**Response Body**

.. code-block:: json

    {
        "limit": {
            "resource_name": "volume",
            "region_id": null,
            "links": {
                "self": "http://10.3.150.25/identity/v3/limits/25a04c7a065c430590881c646cdcdd58"
            },
            "service_id": "9408080f1970482aa0e38bc2d4ea34b7",
            "project_id": null,
            "domain_id": "3a705b9f56bb439381b43c4fe59dccce",
            "id": "25a04c7a065c430590881c646cdcdd58",
            "resource_limit": 11,
            "description": null
        }
    }

**Request:** ``GET /limits``

The response body is similar with GET /limits. Add the `domain_id` filter to
query the specified limits.

Model change
------------
Now both `flat` model and `strict-two-level` model are only for project level.

Non hierarchical model
^^^^^^^^^^^^^^^^^^^^^^
Like Flat model, it is non hierarchical. So domain level limit in this kind of
model is non hierarchical as well.

Hierarchical model
^^^^^^^^^^^^^^^^^^
Like strict-two-level model, For this kind of model, Every project's limit in
a domain should not be larger than the domain's limit.

Of course, The domain level be considered as one level In strict-two-level
model currently, the depth of project should not greater than
2. Like::

  Project(level-1)
       |
  Project(level-2)

Once domain level limit is supported, the
structure will be::

  Domain(level-1)
    |
  Project(level-2)


Error Handler
-------------

The behavior for error handler will not be changed, It'll keep the same with
the enforcement model. See the detail in the `enforcement`_ part of
strict-two-level-enforcement-model spec.

.. _`enforcement`: http://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/strict-two-level-enforcement-model.html#enforcement-diagrams

Backward compatibility
----------------------
The unified limit feature is still experimental in Keystone and there is no
customer currently. So there is not need to consider the backward compatibility
now.

Alternatives
------------

Database change
^^^^^^^^^^^^^^^
We can reuse the `project_id` column to store `domain_id`. At database layer,
`domain_id` can be treat the same as `project_id`.

But this change would make the code much complex. Some code for dealing with
`project_id` and `domain_id` would be added of which the user experience is not
good.

Hierarchical model
^^^^^^^^^^^^^^^^^^
For hierarchical model, we can treat domain level as the top level which does
not consume the project level depth. Like::

  Domain
    |
  Project-level1
    |
  Project-level2


But if so, the depth will be more than 2 which will break strict two concept.
The strict-two-level-enforcement-model spec has a good `explanation`_

.. _explanation: http://specs.openstack.org/openstack/keystone-specs/specs/keystone/rocky/strict-two-level-enforcement-model.html#optimized-usage-calculation

Security Impact
---------------

N/A

Notifications Impact
--------------------

N/A


Other End User Impact
---------------------

This is a cloud admin feature. It doesn't impact end user through APIs. If the
end user is domain level, he may have limitation for cloud resource then.


Performance Impact
------------------

N/A

Other Deployer Impact
---------------------

N/A

Developer Impact
----------------

N/A

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  wangxiyuan<wangxiyuan@huawei.com>

Work Items
----------

* Update database schema
* Update POST/GET /limits /limits/{limit_id} APIs.
* Update limit model check function.
* Update related docs.


Dependencies
============

N/A


Documentation Impact
====================

Domain level limit support should be documented in admin guide.


References
==========

* OpenStack Public Cloud WG `requirement`_

.. _requirement: https://bugs.launchpad.net/openstack-publiccloud-wg/+bug/1771581

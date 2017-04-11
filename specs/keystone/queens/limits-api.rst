..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================
Unified Limits API
==================

Detailed specification for the work necessary to associate resources limits to
projects.

`bp unified-limits <https://blueprints.launchpad.net/keystone/+spec/unified-limits>`_

Problem Description
===================

Today, resource quotas are maintained outside Keystone in each project, such as
Nova, Cinder or others. This leads to a problem where there is no strong
relationship between projects and their resources. For example, when a user
sets a quota in Nova like "project A can create 10 virtual machines", but if
project A doesn't exist in Keystone, Nova will still create the quota. If the
project A is deleted in Keystone, Nova doesn't know it and will still leave the
quota there. This problem is only exacerbated when dealing with project
hierarchies.

More information on this problem and the overall approach can be found in a
separate high-level `specification <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/ongoing/unified-limits.html>`_.

Proposed Change
===============

An interface to create, read, update, delete limit definitions. It includes two
kinds of Limits: ``Registered Limits`` and ``Project Limits``.
``Registered Limits`` are the defaults limits. You can't set a project limit on
something that's not a registered limit.
``Project Limits`` are the limits that override registered limits for each
project.

.. note::

   This spec doesn't talk about the usage check of limits. It is up to the
   consuming services to enforce usage.

Registered Limits
-----------------

The following API calls are specific to the management of Registered Limits.

Create Registered Limits
------------------------

Limits are registered with the Registered Limits endpoint. Because
limits definitions will often be created in bulk, this supports
sending a number of limit definitions at once. This is an admin only
action.

**Request:** ``POST /registered-limits``

**Request Parameters**

* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in. If the
  ``region_id`` is specified, it should be keep the same with the consuming
  service's. If not, Keystone will leave it empty like endpoint does.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Request Body**

.. code-block:: json

    {
        "limits": [
            {
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "default_limit": 10
            },
            {
                "service_id": "77232e5107074dfe801657000348e8c9",
                "resource_name": "ram_mb",
                "default_limit": 20480
            },
        ]
    }

**Response**

A full list of all registered limits.

**Response Parameters**

* ``id`` - The unique uuid for each registered limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Response Code**

* 200 - OK
* 400 - Bad Request - if dependent resources do not exist
* 403 - Forbidden - if the user is not authorized to create a registered limit
* 409 - Limits that already exist

**Response Body:**

.. code-block:: json

    {
        "limits": [
            {
                "id": "0681e10c01c044d78ef8e5cb592c6446",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "default_limit": 10
            },
            {
                "id": "1dc633fe5acd4182b63f68c9cc8e768a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "resource_name": "ram_mb",
                "default_limit": 20480
            },
            {
                "id": "5c4182eb45304cb3ac89030b19ab5a81",
                "service_id": "ae22fb0dfbd34464bf67e758977f4839",
                "region_id": "regionOne",
                "resource_name": "storage_gb",
                "default_limit": 20
            },
        ]
    }

Update Registered Limits
------------------------

Update is done similar to a POST however just the limits that you wish
to override are included. If the ``service_id``, ``region_id``, or
``resource_name`` doesn't already exist, an error is thrown.

**Request:** ``PUT /registered-limits``

**Request Parameters**

* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region_id that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Request Body**

.. code-block:: json

    {
        "limits":[
            {
                "id": "1dc633fe5acd4182b63f68c9cc8e768a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "resource_name": "ram_mb",
                "default_limit": 10240
            }
        ]
    }

**Response:**

A full list of all limits. That allows for double checking one's work.

**Response Code:**

* 200 - OK
* 400 - Bad Request - if dependent resources do not exist
* 403 - Forbidden - if the user is not authorized to update a registered limit

**Response Parameters**

* ``id`` - The unique uuid for each registered limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "limits": [
            {
                "id": "0681e10c01c044d78ef8e5cb592c6446",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "default_limit": 10
            },
            {
                "id": "1dc633fe5acd4182b63f68c9cc8e768a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "resource_name": "ram_mb",
                "default_limit": 10240
            },
            {
                "id": "5c4182eb45304cb3ac89030b19ab5a81",
                "service_id": "ae22fb0dfbd34464bf67e758977f4839",
                "region_id": "regionOne",
                "resource_name": "storage_gb",
                "default_limit": 20
            },
        ]
    }

List all Registered Limits
--------------------------

Registered limits can be read by anyone with a valid token.

**Request:** ``GET /registered-limits``

**Request filter:**

Registered limits will also support filters to make it easier to see
just a subset. ``service_id``, ``region_id``, and ``resource_name``
will all be valid search parameters.

**Response:**

A full list of all registered limits.

**Response Code:**

* 200 - OK
* 403 - Forbidden - if the user is not authorized to list registered limits

**Response Parameters**

* ``id`` - The unique uuid for each registered limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "limits": [
            {
                "id": "0681e10c01c044d78ef8e5cb592c6446",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "default_limit": 10
            },
            {
                "id": "1dc633fe5acd4182b63f68c9cc8e768a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "resource_name": "ram_mb",
                "default_limit": 20480
            },
            {
                "id": "5c4182eb45304cb3ac89030b19ab5a81",
                "service_id": "ae22fb0dfbd34464bf67e758977f4839",
                "region_id": "regionOne",
                "resource_name": "storage_gb",
                "default_limit": 20
            },
        ]
    }

Show a Registered Limits
------------------------

Registered limits can be read by anyone with a valid token.

**Request:** ``GET /registered-limits/{registered-limits-id}``

**Request Parameters**

* ``registered-limits-id`` - The id for the specified registered limit.

**Response:**

The specified registered limit.

**Response Code:**

* 200 - OK
* 403 - Forbidden - if the user is not authorized to retrieve a registered
  limit
* 404 - Not Found - if the requested registered limit does not exist

**Response Parameters**

* ``id`` - The unique uuid for each registered limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``default_limit`` - The default limit for all projects to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "id": "0681e10c01c044d78ef8e5cb592c6446",
        "service_id": "77232e5107074dfe801657000348e8c9",
        "region_id": "regionOne",
        "resource_name": "cores",
        "default_limit": 10
    }

Delete a Registered Limit
-------------------------

**Request:** ``DELETE /registered-limits/{registered-limits-id}``

**Request Parameters**

* ``registered-limits-id`` - The id for the specified registered limit.

**Response:**

No content.

**Response Code:**

* 204 - No Content
* 403 - Forbidden - if the user is not authorized to delete a registered limit
* 404 - Not Found - if the requested registered limit does not exist


Project Limits
--------------

The following API calls are specific to the management of Project Limits. They
are project administrator only (system admin as well) APIs.

.. note::

   The initial implementation will only support a "flat" hierarchical model. In
   this model, the limits associated to a project will be validated as a flat
   structure. This means limits won't be enforced or validated according to
   the parents, childred, or peers of the project. All limits will be
   independent of those relationships. This is referred to as a "flat"
   enforcement model. Future work will elaborate on more complex enforcement
   models that understand project hierarchies.

Create Project Limits
---------------------

Overriding Registered Limits with Project Limits.

**Request:** ``POST /limits``

**Request Parameters**

* ``project_id (optional)`` - The project which assume the limit. If omit,
  Keystone will get the project_id from token (context).
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in. It
  should use same ``region_id`` of the registered limit which will be
  overridden.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Request Body**

.. code-block:: json

    {
        "limits":[
            {
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "ram_mb",
                "resource_limit": 10240,
            },
            {
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "resource_limit": 10,
            },
        ]
    }

**Response:**

We return the entire limits structure, including defaults without overrides.

**Response Code:**

* 200 - OK
* 400 - Bad Request - if dependent resources do not exist
* 403 - Forbidden - if the user is not authorized to change the limit for that
  project
* 409 - Limits that already exist

**Response Parameters**

* ``id`` - The id for the specified limit.
* ``project_id`` - The project which assume the limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "limits":[
            {
                "id": "aaab50e9c36f4a84bab98dfc117c9836",
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "ram_mb",
                "resource_limit": 10240,
            },
            {
                "id": "e08fcb2756be48e387e821bd79e29538",
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "resource_limit": 10,
            },
        ]
    }

Update Project Limits
---------------------

Update Project Limits. Once the project limit is created,  The only property
that can be changed is ``resource_limit``.

**Request:** ``PUT /limits``

**Request Parameters:**

* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Request Body:**

.. code-block:: json

    {
        "limits":[
            {
                "id": "aaab50e9c36f4a84bab98dfc117c9836",
                "resource_limit": 5120,
            },
            {
                "id": "e08fcb2756be48e387e821bd79e29538",
                "resource_limit": 5,
            },
        ]
    }

**Response:**

We return the entire limits structure, including defaults without overrides.

**Response Code:**

* 200 - OK
* 400 - Bad Request - if registered limit matching the resource name or the
  project limit with the given ID do not exist
* 403 - Forbidden - if the user is not authorized to change the limit for that
  project

**Response Parameters**

* ``id`` - The id for the specified limit.
* ``project_id`` - The project which assume the limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "limits":[
            {
                "id": "aaab50e9c36f4a84bab98dfc117c9836",
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "ram_mb",
                "resource_limit": 5120,
            },
            {
                "id": "e08fcb2756be48e387e821bd79e29538",
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "resource_limit": 5,
            },
        ]
    }

List Project Limits
-------------------

**Request:** ``GET /limits``


**Request filter:**

* ``project_id`` - Only used for cloud admin to filter limits with the
  specified project_id. Project admin can only list the limits for their own
  projects.

limits will also support filters to make it easier to see
just a subset. ``service_id``, ``region_id``, and ``resource_name``
will all be valid search parameters.

**Response:**

A list of all limits in a project.

**Response Code:**

* 200 - OK
* 403 - Forbidden - if the user is not authorized to list limits for that
  project

**Response Parameters**

* ``id`` - The id for the specified limit.
* ``project_id`` - The project which assume the limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "limits":[
            {
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "ram_mb",
                "resource_limit": 10240,
            },
            {
                "project_id": "95541dbfaa054cab86510e0d0a87896a",
                "service_id": "77232e5107074dfe801657000348e8c9",
                "region_id": "regionOne",
                "resource_name": "cores",
                "resource_limit": 10,
            },
        ]
    }

Show a Project Limit
--------------------

**Request:** ``GET /limits/{limit-id}``

**Request Parameters:**

* ``limit-id`` - The id for the specified limit.

**Response:**

The detail of the specified limit.

**Response Code:**

* 200 - OK
* 403 - Forbidden - if the user is not authorized to retrieve that project
  limit
* 404 - Not Found - if the requested project limit does not exist

**Response Parameters**

* ``id`` - The id for the specified limit.
* ``project_id`` - The project which assume the limit.
* ``service_id`` - The service that is responsible for the resource, which
  should match a service in the service catalog.
* ``region_id (optional)`` - The region that the service is hosted in.
* ``resource_name`` - The name of the resource, which should be unique compared
  to other resource names for the same service and region.
* ``resource_limit`` - The override limit for the project to assume for that
  resource.

**Response Body:**

.. code-block:: json

    {
        "project_id": "95541dbfaa054cab86510e0d0a87896a"
        "service_id": "77232e5107074dfe801657000348e8c9",
        "region_id": "regionOne",
        "resource_name": "ram_mb",
        "resource_limit": 10240,
    }

Delete a Project Limit
----------------------

**Request:** ``DELETE /limits/{limit-id}``

**Request Parameters**

* ``limit-id`` - The id for the specified limit.

**Response:**

No content

**Response Code:**

* 204 - No Content
* 403 - Forbidden - if the user is not authorized to delete a limit for that
  project
* 404 - Not Found - if the requested project limit does not exist

Flat Hierarchy Enforcement
--------------------------

Keystone supports hierarchical multi-tenancy, where projects can be grouped
into tree structures and have parents, siblings, and children. It's possible to
think of various ways where a project limit interacts differently depending on
the limits of other projects in the tree. The initial implementation of project
limits documented in this specification is going to account for a flat
structure. This means the limit information and validation does **not** account
for other projects in the hierarchy. Each project has it's own limit.

Assume project ``P`` is a child of project ``F``, which is a child of project
``A``. A default is set on a limit, all projects get that effective default.

Assuming we have a default limit of 10

.. blockdiag::

   blockdiag {
      orientation = portrait;

      A [label="A (10)"];
      F [label="F (10)"];
      P [label="P (10)"];
   }

And then we ``UPDATE LIMIT on A to 20``

.. blockdiag::

   blockdiag {
      orientation = portrait;

      A [label="A (20)", textcolor = "#FF0000"];
      F [label="F (10)"];
      P [label="P (10)"];
   }

Or we can ``UPDATE LIMIT on P to 30``

.. blockdiag::

   blockdiag {
      orientation = portrait;

      A [label="A (20)"];
      F [label="F (10)"];
      P [label="P (30)", textcolor = "#FF0000"];
   }

This is allowed with flat enforcement because the hierarchy is not taken into
consideration during limit validation. In the future, we will introduce a model
that has the ability to validate limits with respect to project hierarchies.
It is important to note that switching between enforcement models will require
extremely careful planning and possibly lead to API changes depending on the
request being made and the new enforcement model. Deployments need to be aware
of this, understand the ramifications of switching enforcement models, and the
impacts it can have on existing limits.

Keystone will also expose a ``GET /limits-model`` endpoint that is responsible
for returning the enforcement model selected by the deployment. This is key to
allowing discoverable limit models and perserving interoperability between
OpenStack deployments with different enforcement models.

Alternatives
------------

One alternative that's already been taken by at least one project is to attempt
to implement hierarchical quotas in the service itself. Since understanding the
hierarchy can be confusing, not duplicating that logic is what lead us to this
approach, which keeps the limit closely associated to the hierarchy.

Another alternative is that we can add limits inside projects. The API will be
like /projects/{project_id}/limits/{limit_id}. These APIs has ways of showing
a project hierarchy already. In this way, we can reuse it easily.

Security Impact
---------------

The enforcement and validation of limits targeted in this work is specific to
flat hierarchies. This means that limits are associated to project
independently, regardless of parent, children, or peer projects. For example,
assume project ``alpha`` is a parent of projects ``bravo`` and ``charlie``. A
flat hierarchy would allow ``bravo`` to have a limit of 10 instances while
``charlie`` and ``alpha`` may only have a limit of 5 instances. From the
perspective of a project hierarchy, this may feel unintuitive. This is the
first enforcement model implementation and once we build knowledge and collect
usage feedback, there will be an effort to develop more sophisticated
enforcement models that account for project hierarchies.

Registered limits should be considered public information and discoverable.
Project limits should be available to members of the project. A user with a
role on project ``alpha`` should be able to list limits for the project, but
not for ``bravo`` or ``charlie``. This case will become more complicated in the
future when we start developing enforcement models that account for
hierarchies.

When more complicated models are introduced, we will need a way to provide
sufficient information to the user to allow them to understand why a limit
update has failed or why a resource request brings them over quota without
divulging too much information about related projects. This will not need to be
addressed with this initial "flat" implementation.

Notifications Impact
--------------------

Registered Limits and Project Limits should be subject to the same
notifications as other resources in keystone.

Other End User Impact
---------------------

None. End users will be able to query keystone for limit information. This
improves usability because they can see what the limit is and gather more
information when requesting help from an administrator.

Performance Impact
------------------

The internal performance impact of the initial flat hierarchy design should be
negligible. This will likely become more complicated once development for
hierarchical enforcement models starts (e.g. calculating limits of a project
with respect to its parent(s), children, and peers). Keystone will then have to
compute more complicated limit structure.

Other services will be required to make additional calls to keystone to
retrieve limit information in order to do quota enforcement. This will add some
overhead to the overall performance of the API call.

It is also worth noting that both Registered Limits and Project Limits are not
expected to change frequently. This means the data is safe to cache for some
period of time. Caching will be implemented internally to keystone, similar to
how keystone caches responses for other resources. But, caching can also be
done client-side to avoid making frequent calls to keystone for relatively
static limit information.

Other Deployer Impact
---------------------

Deployments looking to have Registered Limits and Project Limits in keystone
will have to set that up at installation time. This creates an extra step for
operators, similar to how they register services in the service catalog.

Developer Impact
----------------

Developers from other projects will likely have the following questions:

* What the difference between a Registered Limits and Project Limit?
* What information is relayed in the limit?
* How do I enforce usage based on the information about a limit?
* Is there a library to do this for me?

There are a lot of things we'll have to make sure we communicate to
developers looking to implement hierarchical quotas. Keystone's
really just the information point here. We need to be available on
the other side to help them consume that information.

These questions, among others, will likely have to be answered in developer
documentation within keystone.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * wangxiyuan <wangxiyuan@huawei.com> wxy

Other contributors:

  * Lance Bragstad <lbragstad@gmail.com> lbragstad
  * Colleen Murphy <colleen@gazlene.net> cmurphy

Work Items
----------

1. Implement unified limits, add the new APIs mentioned above to Keystone.
2. Implement client support for unified limits.
3. Document limit models, Document unified limits, add related developer and
   user DOC.

The `epic <https://trello.com/c/OMtxcBeh/16-unified-limits>`_ tracking this
work can be found in keystone's Queens roadmap.

.. note::

   Make sure the APIs are generic enough so that we can support more quota
   model in the future.

Dependencies
============

None

Documentation Impact
====================

The usage of the new limit APIs should be addressed.

References
==========

High-level `overview <http://specs.openstack.org/openstack/keystone-specs/specs/keystone/ongoing/unified-limits.html>`_
of limits.

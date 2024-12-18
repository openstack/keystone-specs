..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Strict Two-Level Limits Enforcement Model
=========================================

This specification describes the behaviors and use-cases of a strict
enforcement model for limits associated to resources in a hierarchical project
structure.

`bp strict-two-level-model  <https://blueprints.launchpad.net/keystone/+spec/strict-two-level-model>`_

Problem Description
===================

The unified limit `specification`_ and implementation, introduced in the Queens
release, ignore all details about project structure. It's only enforcement
model is `flat`_. This means limits associated to any part of the tree are not
validated against each other.

.. _specification: http://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/limits-api.html
.. _flat: https://docs.openstack.org/keystone/latest/admin/unified-limits.html#flat

Proposed Change
===============

This specification goes through the details for a strict two-level hierarchical
enforcement model.

Use Cases
---------

* As an operator, I want to be able to set the limit for a top-level project
  and ensure its usage never exceeds that limit, resulting in strict usage
* As a user responsible for managing limits across projects, I want to be able
  to set limits across child projects in a way that is flexible enough to allow
  resources to flow between projects under a top-level project

These use cases were mentioned on the mailing list in an early `discussion`_
about unified limits.

.. _discussion: http://lists.openstack.org/pipermail/openstack-dev/2017-February/111999.html

Model Behaviors
---------------

This model:

* requires project hierarchy never exceeds a depth of two, meaning hierarchies
  are limited to parent and child relationships
* requires each tree have a single parent, or tree root
* allows parents, or tree roots, to have any number of children
* allows quota overcommit, i.e. the aggregate quota limit (not usage) may
  exceed the limit of the parent. Overcommit and user-experience related to
  overcommit is a leading factor for the strict two-level hierarchy.
* does not directly solve sharing data across endpoints, e.g.
  each nova would not be aware of the other nova's quota consumption meaning
  a user could consume the full amount of quota on each endpoint.

This model implements limit validation in keystone that:

* allows the sum of all child limits to exceed the limit of the parent, or tree
  root
* disallows a child limit from exceeding the parent limit
* assumes registered limits as the default for projects that are not given a
  project-specific override

This model is consumed by ``oslo.limit`` in a way that:

* requires services responsible for resources to implement a usage callback for
  ``oslo.limit`` to use to calculate usage for the project tree
* requires that usage be calculated on every request

The ``oslo.limit`` library will enforce the model such that the resource usage
sum across the entire tree cannot exceed the resource limit set by the parent.

This model is called a ``strict-two-level`` enforcement model. It is `strict`
because the usage of a resource across the entire tree can never exceed the
parent limit. It is considered a `two-level` model because it only assumes to
work on project hierarchies of two or less.

Enforcement Diagrams
--------------------

The following diagrams illustrate the above behaviors, using projects named
``A``, ``B``, ``C``, and ``D``. Assume the resource in question is ``cores``,
and the default registered limit for ``cores`` is 10.  The labels in the
diagrams below use shorthand notation for `limit` and `usage` as `l` and `u`,
respectively.

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=4)"];
      B [label="B (u=0)"];
      C [label="C (u=0)"];
   }

Technically, both ``B`` and ``C`` can use up to 8 ``cores`` each, resulting in:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=4)"];
      B [label="B (u=8)", fontcolor = "#00af00"];
      C [label="C (u=8)", fontcolor = "#00af00"];
   }

If ``A`` attempts to claim two more ``cores``, the usage check will fail
because ``oslo.limit`` will fetch the hierarchy from keystone and check the
usage of each project in the hierarchy by using the callback provided by the
service to see that total usage of ``A``, ``B``, and ``C`` is equal to the
limit of the tree, set by ``A.limit``.

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=6)", fontcolor = "#FF0000"];
      B [label="B (u=8)"];
      C [label="C (u=8)"];
   }

Despite the usage of the tree being equal to the limit, we can still add
children to the tree:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      A -> D;

      A [label="A (l=20, u=4)"];
      B [label="B (u=8)"];
      C [label="C (u=8)"];
      D [label="D (u=0)", fontcolor = "#00af00"];
   }

Even though the project can be created, the current usage of cores across the
tree prevents ``D`` from claiming any ``cores``:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      A -> D;

      A [label="A (l=20, u=4)"];
      B [label="B (u=8)"];
      C [label="C (u=8)"];
      D [label="D (u=2)", fontcolor = "#FF0000"];
   }

Creating a grandchild of project ``A`` is forbidden because it violates the
two-level hierarchy constraint. This is a fundamental contraint of this design
because it provides a very clear escalation path. When a request fails because
the tree limit has been exceeded, a user has all the information they need to
provide meaningful context in a support ticket (e.g. their project ID and the
parent project ID). An administrator of project ``A`` should be able to
reshuffle usage accordingly. A system administrator should be able to as well.
Providing this information in tree structures with more than a depth of two is
much harder, but may be implemented with a separate model.

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      C -> D;

      A [label="A (l=20, u=4)"];
      B [label="B (u=8)"];
      C [label="C (u=8)"];
      D [label="D (u=0)", fontcolor = "#FF0000"];
   }

Granting ``B`` the ability to claim more cores can be done by giving ``B`` a
project-specific override for ``cores``:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=4)"];
      B [label="B (l=12, u=8)", fontcolor = "#00af00"];
      C [label="C (u=8)"];
   }

Note that regardless of this update, any subsequent requests to claim more
``cores`` in the tree will be forbidden since the usage is equal to the limit
of the ``A``. If ``cores`` are released from ``A`` and ``C``, ``B`` can claim
them:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=2)", fontcolor = "#00af00"];
      B [label="B (l=12, u=8)"];
      C [label="C (u=6)", fontcolor = "#00af00"];
   }

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=2)"];
      B [label="B (l=12, u=12)", fontcolor = "#00af00"];
      C [label="C (u=6)"];
   }

While ``C`` is still under its default allocation of 10 ``cores``, it won't be
able to claim any more ``cores`` because the total usage of the tree is equal
to the limit of ``A``, thus preventing ``C`` from reclaiming the ``cores`` it
had:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=2)"];
      B [label="B (l=12, u=12)"];
      C [label="C (u=8)", fontcolor = "#FF0000"];
   }

Creating or updating a project with a limit that exceeds the limit of ``A`` is
forbidden. Even though it is possible for the sum of all limits under ``A`` to
exceed the limit of ``A``, the total usage is capped at ``A.limit``. Allowing
children to have explicit overrides greater than the limit of the parent would
result in strange user experience and be misleading since the total usage of
the tree would be capped at the limit of the parent:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A (l=20, u=0)"];
      B [label="B (l=30, u=0)", fontcolor = "#FF0000"];
      C [label="C (u=0)"];
   }

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      A -> D;

      A [label="A (l=20, u=0)"];
      B [label="B (u=0)"];
      C [label="C (u=0)"];
      D [label="D (l=30, u=0)", fontcolor = "#FF0000"];
   }

Finally, let's still assume the default registered limit for ``cores`` is 10,
but we're going to create project ``A`` with a limit of 6.

.. graphviz::

   digraph {
      node [shape=box]

      A;

      A [label="A (l=6, u=0)", fontcolor = "#00af00"];
   }

When we create project ``B``, which is a child of project ``A``, the limit API
should ensure that project ``B`` doesn't assume the default of 10. Instead, we
should obey the parent's limit since no single child limit should exceed the
limit of the parent:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;

      A [label="A (l=6, u=0)"];
      B [label="B (l=6, u=0)", fontcolor = "#00af00"];
   }

This behavior should be consistent regardless of the number of children added
under project ``A``.

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      A -> D;

      A [label="A (l=6, u=0)"];
      B [label="B (l=6, u=0)"];
      C [label="C (l=6, u=0)", fontcolor = "#00af00"];
      D [label="D (l=6, u=0)", fontcolor = "#00af00"];
   }

Creating limit overrides while creating projects seems counter-productive given
the whole purpose of a registered default, but it also seems unlikely to
throttle a parent project by specifying it's default to be less than a
registered default. This behavior maintains consistency with the requirement
that the sum of all child limits may exceed the parent limit, but the limit of
any one child may not.

Proposed Server Changes
-----------------------

Keystone will need to encapsulate this logic into a new enforcement model.
Ideally, this enforcement model can be called from within the unified limit API
to validate limits before writing them to the backend.

If keystone is configured to use the ``strict-two-level`` enforcement model and
current project structure within keystone violates the two-level project
constraint, keystone should fail to start. On start-up, keystone will scan the
database to ensure that all projects don't exceed two levels of hierarchy and
that ``keystone.conf [DEFAULT] max_project_tree_depth = 2``. If either
condition fails, keystone will report an appropriate error message and refuse
to start.

To aid operators, we can develop a ``keystone-manage`` command, to check the
hierarchical structure of the projects in the deployment and warn operators if
keystone is going to fail to start. This gives operators the ability to check
and fix their project hierarchy before they deploy keystone with the new model.
This clearly communicates a set project structure to operators at run time.

Proposed Library Changes & Consumption
--------------------------------------

The ``oslo.limit`` library is going to have to know when to enforce usage based
on the ``strict-two-level`` model. It can ask for the current model by querying
the limit API directly:

**Request:** `GET /v3/limits/model`

**Response**

* 200 - OK
* 401 - OK

**Response Body**

.. code:: json

   {
       "model": {
           "name": "strict-two-level",
           "description": "Strict usage enforcement for parent/child relationships."
        }
   }

The library should expose an object for claims and a context manager so that
consuming services can make the following call from within their API business
logic:

.. code::

   from oslo_limit import limit
   LIMIT_ENFORCER = limit.Enforcer()

    def create_foobar(self, context, foobar):

        claim = limit.ProjectClaim('foobars', context.project_id, quantity=1)
        callback = self.get_resource_usage_for_project
        with limit.Enforcer(claim, callback=callback):
            driver.create_foobar(foobar)


In the above code example, the service builds a ``ProjectClaim`` object that
describes the resource being consumed and the project. The ``claim`` is then
passed to an ``oslo.limit`` context manager and supplimented with a callback
method from the service. The service's callback method is responsible for
calculating resource usage per project. The ``oslo.limit`` library can use the
``project_id`` from the context object to get the limit information from
keystone and calculate usage across the project tree with the callback. The
usage check for the project hierarchy will be executed when the context manager
is instantiated or executing ``__enter__``. By default, exiting the context
manager will verify that the usage was not exceeded by another request,
protecting from race conditions across requests. This can be disabled explicity
using the following::

   from oslo_limit import limit
   LIMIT_ENFORCER = limit.Enforcer()

    def create_foobar(self, context, foobar):

        claim = limit.ProjectClaim('foobars', context.project_id, quantity=1)
        callback = self.get_resource_usage_for_project
        with limit.Enforcer(claim, callback=callback, verify=False):
            driver.create_foobar(foobar)

Fetching project hierarchy
^^^^^^^^^^^^^^^^^^^^^^^^^^

The (current) default policy prevents users with a member role on a project
from retrieving the entire project hierarchy. The library that needs the
hierarchy to calculate usage must call the API as a project administrator or
use a service user token. This API is used for both *operators* and
*oslo.limit*.

**Request:** ``GET /limits?show_hierarchy=true``

**Request filter**

* ``show_hierachy`` - Whether to show the hierarchy project limit or not.

**Response:**

A list of the hierarchy project limits.

**Response Code:**

* 200 - OK
* 404 - Not Found

**Response Body:**

.. code:: json

    {
        "limits":[
            {
                "id": "c1403b468a9443dcabf7a388234f3f68",
                "service_id": "e02156d4fa704d02ac11de4ddba81044",
                "region_id": null,
                "resource_name": "ram_mb",
                "resource_limit": 20480,
                "project_id": "fba8184f0b8a454da28a80f54d80b869",
                "limits": [
                    {
                        "id": "7842e3ff904b48d89191e9b37c2d29af",
                        "project_id": "f7120b7c7efb4c2c8859441eafaa0c0f",
                        "region_id": null,
                        "resource_limit": 10240,
                        "resource_name": "ram_mb",
                        "service_id": "e02156d4fa704d02ac11de4ddba81044"
                    },
                    {
                        "id": "d2a6ebbc5b0141178c07951a10ff547c",
                        "project_id": "443aed1062884dd38cd3893089c3f109",
                        "region_id": null,
                        "resource_limit": 5120,
                        "resource_name": "ram_mb",
                        "service_id": "e02156d4fa704d02ac11de4ddba81044"
                    },
                    {
                        "id": "f8b7f4da96854c4cafe3d985acc5110f",
                        "project_id": "ca7e4b4cd7b849febb34f6cc137089d0",
                        "region_id": null,
                        "resource_limit": 2560,
                        "resource_name": "ram_mb",
                        "service_id": "e02156d4fa704d02ac11de4ddba81044"
                    }
                ]
            }
        ]
    }


The above is an example response given the following diagram, where the default
registered limit for ``ram_mb`` is 2560, which applies to ``D``.

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      A -> D;

      A [label="A (l=20480)"];
      B [label="B (l=10240)"];
      C [label="C (l=5120)"];
      D [label="D (l=undefined)"];
   }

Alternatives
------------

Stick with a flat enforcement model, requiring operators to manually implement
hierarchical limit knowledge.

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

Usage Caching
^^^^^^^^^^^^^

Performance of this model is expected to be sub-optimal in comparison to flat
enforcement. The main factor contributing to expected performance loss is the
calculation of usage for the tree. The ``oslo.limit`` library will need to
calculate the usage for every project in the tree in order to provide an answer
to the service regarding the request.

One possible solution to mitigate performance concerns here would be to
calculate usage for projects in parallel.

Limit Caching
^^^^^^^^^^^^^

Other services will be required to make additional calls to keystone to
retrieve limit information in order to do quota enforcement. This will add some
overhead to the overall performance of the API call because it requires a
round-trip to keystone.

It is also worth noting that both Registered Limits and Project Limits are not
expected to change frequently. This means that the limit data could possibly be
cached client-side in ``oslo.limit``. However, this introduces concerns about
limit invalidation. Consider the following example:

* client-side cache TTL is set to one hour for limit information from keystone
* one minute later, an administrator decreases the limit for ``cores`` on a
  particular project
* two minutes later, a user makes a request to create an instance in the
  project that should bring them over the limit just set by the administrator
* due to client-side caching, the service considers the project within it's
  limit and allows the instance to be created
* the current usage is out of sync with the limit set by the administrator and
  the service won't realize this until the TTL expires in another 57 minutes

Client-side caching is going to be a very specific case that needs to be
handled carefully because cache invalidation strategies are going to be
distributed across services. One possible mitigation would be for client-side
caching and keystone to share the same cache instance, making it easier to
perform cache invalidation. Conversely, this raises the operational bar for
administrators and requires assumptions about underlying infrastructure.

Until we know we can make those types of assumptions or find an alternative
solution for cache invalidation, client-side caching should be avoided to
prevent situations like what was described above. We should error on the side
of accuracy when retrieving limit information.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

The enforcement library ``oslo.limit`` should be implemented based on the
enforcement model implemented in keystone.

The consuming component (e.g. nova, neutron, cinder, etc..)should add the new
way to fetching quota limit from keystone in the future.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * wangxiyuan <wangxiyuan@huawei.com> wxy
  * Lance Bragstad <lbragstad@gmail.com> lbragstad

Other contributors:


Work Items
----------

* Add the new API ``GET /limits/model``
* Abstract limit validation into a model object
* Implement a new limit model for ``strict-two-level``
* Implement ``strict-two-level`` enforcement in ``oslo.limit``
* Add the new ``show_hierachy`` parameter for limits.
* Add keystone client support for limits.

Future work
-----------

Limit and Usage Awareness Across Endpoints
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``oslo.limit`` and keystone server can be enhanced to utilize ``etcd`` (or
other shared data store) to represent limit data and cross-endpoint
quota-usage. This falls out of scope for this particular specification.  It
should be noted that the model should be able to consume the data from whatever
store is used, not restricted to a local-only datastore.

Optimized Usage Calculation
^^^^^^^^^^^^^^^^^^^^^^^^^^^

During the design of this enforcement model, various parties mentioned
performance-related concerns when employing this model for trees with many
projects. For example, calculating usage for ``cores`` across hundreds or
thousands of projects. Consider the following tree structure:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;

      A [label="A"];
      B [label="B"];
      C [label="C"];
   }

Consider that each project not only has the concept of ``usage`` and ``limit``,
but also something called an ``aggregate``. An ``aggregate`` is the sum of a
projects ``usage`` and all ``aggregrate`` counts of its children.

For example, when claiming two ``cores`` on ``C``, ``C.usage=2`` and
``C.aggregate=2``. The tree root, ``A``, is also updated in this case where
``A.aggregate=2``. When a subsequent claim is made on ``B`` updating its usage
to ``B.usage=2``, the usage calculation only needs to check the ``aggregate``
usage property of the parent project, or the project tree.

This simplifies the usage calculation process by only having to query the
parent, or tree root, for it's aggregate usage. As opposed to querying each
project for it's usage and sum the result of each aggregate stored for the
parent.

The following illustrates a more extreme example:

.. graphviz::

   digraph {
      node [shape=box]

      A -> B;
      A -> C;
      B -> D;
      B -> E;

      A [label="A"];
      B [label="B"];
      C [label="C"];
      D [label="D"];
      E [label="E"];
   }

Let's assume each project has ``usage=0`` and ``limit=10``. The following might
be a possible scenario: When claiming
resources on ``D.usage=4``

* SET ``D.usage=4 AND D.aggregate=4``
* SET ``B.aggregate=4``, since ``B.usage=0`` currently
* SET ``A.aggregate=4``, since ``A.usage=0`` currently
* SET ``C.usage=6 AND C.aggregate=6``
* SET ``A.aggregate=10``, since ``A.usage=0`` still
* SET ``E.usage=2`` fails

The last step in the flow fails because the entire tree is already at limit
capacity for ``cores`` once ``C`` finalizes its claim. The advantage is that we
only need to ask for ``A.aggregate`` in order to calculate usage when ``E``
attempts to make its claim, since finalized claims "bubble up" the tree.

Note that this requires services to track usage and aggregate usage, raising
the bar for adoption if this is a desired path forward.

Dependencies
============

* Requires API to expose configured limit model (see `bug 1765193`_)
* Abstract model into it's own area of keystone to keep limit CRUD isolated
  from enforcement model

.. _bug 1765193: https://launchpad.net/bugs/1765193

Documentation Impact
====================

* The supported limit model and the new enforcement step must be documented.

References
==========

* Queens Unified Limit `specification`_
* High-level `description`_ of unified limits
* Rocky PTG design session `etherpad`_
* Early `review`_ containing model context
* Adam's `blog`_ about quota

.. _specification: http://specs.openstack.org/openstack/keystone-specs/specs/keystone/queens/limits-api.html
.. _description: http://specs.openstack.org/openstack/keystone-specs/specs/keystone/ongoing/unified-limits.html
.. _etherpad: https://etherpad.openstack.org/p/unified-limits-rocky-ptg
.. _review: https://review.openstack.org/#/c/441203/
.. _blog: https://adam.younglogic.com/2018/05/tracking-quota/

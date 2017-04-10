..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
 Unified Limits
================

Many OpenStack services have some mechanism for ensuring that a single
user or project doesn't take over your entire cloud and starve out
other users. This requires every service:

- has some default limits encoded in the project for resources
- keeps track of data relating to project_id / user_id when limits are
  overridden for particular projects/users.
- provides a REST API for changing those, and implements a CLI to do
  the same.
- Ensure that projects / users limits stored are for valid projects /
  users (often not actually done) [#f1]_
- Clean up orphaned data for projects / users that are deleted (often
  not actually done) [#f2]_

That's just on the definition of limits side. Once limits are
defined.

- Count resources that are allocated.
- Enforce that allocated resources don't exceed limits in the system.
- Requests that fail half way through due to going over limit are
  rolled back and do not leave garbage lying around.

And this is only for a flat project structure. The moment a
hierarchical project structure is introduced, all this has to be done
potentially in the context not only of the current project, but taking
into account limits and usage by ancestors, siblings, and children.

What is a limit?
================

A limit is the following information:

- project_id
- API service type (e.g. compute, network, object-storage)
- region
- a resource type (e.g. ram_mb, vcpus, security-groups)
- a default limit (the max resources allowed if no project specific
  override is in place)
- the project specific limit
- user_id (optional, and hopefully can be deprecated)

.. note::

   Some current quota implementations today enforce on ``user_id``
   instead of ``project_id`` for some types of resources. It would be
   ideal if that could be deprecated as part of this transition.

This information has to be accurate and consistent in order for any
quota calculations to be valid. This is even more true if we talk
about the notion of hierarchical quotas, because if data is stored in
limits that violates some basic constraint of the system (for
instance, project A has child project B, and child B vcpu quota is > A
vcpu quota), then all the logic to sensibly calculate usage and quotas
at the service will be buggy and hugely more complex.

Where should this be?
=====================

The current approach to limits puts all this data at the service level
(i.e. in nova in the API database). The service type is known, because
it's the API server in question. The resource types are enforced
because they are in code. The default limits are in config. The per
project allowed are in the database, and project_id/user_id are
accepted unvalidated over the API, and may or may not be valid
projects or users. This gets so wildly complicated for the
hierarchical case. The net effect is that many teams, like the Nova
team, have postponed tackling the problem. [#f3]_

An alternative approach is to move this into keystone. Keystone is the
authoritative source of project_id and user_id information. It is the
authoritative source of any hierarchical relationship between
projects. It is an authority about API services types and regions in a
system.

So all that is required is to teach keystone, in a generic way, about
what resource types and default limits each service type has (possibly
distinct per region), and what project specific overrides might exist.

This could be done with new administrative API calls to keystone to
set these.

- a facility to CRUD a resource type, and default limit, and
  associated service and region. This creates a strong definition of
  all the allowed limits in a cloud, so that limit overrides can be
  strictly validated (no ability to set the limit on ``disc`` when it
  should have been ``disk``)
- a facility to CRUD the limit override for a project / user for a
  resource type. This would be strongly validated on existing service
  types, region, project/user, preregistered resource type, and limit
  being an integer.

It is assumed that resource type default definition would happen
during service install / upgrade through an administrative command in
a similar way to how services and endpoints are defined before
services get used.

In the hierarchical case, when creating/updating/deleting a project
override, the rules for the hierarchical limits would be enforced
before the change is made. We want to guarantee that the hierarchical
limits structure is consistent at all times.

Limits vs. Usage Enforcement
============================

When we talk about a Quota system, we're really talking about two
systems. A system for setting and maintaining limits, the theoretical
maximum usage, and a system for enforcing that usage does not exceed
limits. While they are coupled, they are distinct.

In this proposal, Keystone maintains limits. Keystone's responsibility
is to ensuring that any changes to limits are consistent with related
limits currently stored in Keystone.

Individual services maintain and enforce usage. Services check for
enforcement against the current limits at the time a resource
allocation is requested by a particular user. A usage reflects the
actual allocation of units of a particular resource to a consumer.

Given the above, the following is a possible and legal flow.

* User Jane is in project Baobab
* Project Baobab has a default CPU limit of 20
* User Jane allocated 18 CPUs in project Baobab
* Administrator Kelly sets Project Baobab CPU limit to 10
* User Jane can no longer allocate instance resources in project
  Baobab, until she (or others in the project) have deleted at least 9
  CPUs to get under the new limit

This is the behavior that most administrators want, as it lets them
set the policy of what the future should be when convenient, and
prevent those projects from creating any more resources that would
exceed the limits in question. Note, today some projects prevent
limits from being set lower than existing allocations. That API
behavior will not be honored in this new system. [#f4]_

Users in projects can fix this for themselves by bringing down the
project usage to where there is now headroom. If they don't, at some
point the administrators can more aggressively delete things
themselves.

Common behavior between projects
================================

When we get to an N level project hierarchy, this is going to get
complicated. Doing back of the envelope modeling for different quota
models [#f5]_ shows that there are a lot of different ways this can be
modeled.

Because of this, it's going to be assumed that we're going to need
some common library with both checking that a limit change to an
existing hierarchy is valid, as well as a resource allocation does not
exceed quota. While valid limit checking will be in keystone only, and
quota checking in projects only, having the same algorithms in common
code will ensure that limit changes for `Garbutt Model` are consistent
with quota changes for `Garbutt Model`.

The exact interfaces will need to be hammered out as this gets
implemented.

Access to limits
================

Limit information should be accessed over a REST API call. This is
potentially *extremely* cachable information. Only explicit updates to
Limits, made via Keystone API, will invalidate this
information. Keystone should be able to implement efficient HTTP
caching for this information.

Users in a project will have visibility to all the project limits, as
well as limits in child projects. Depending on the quota system model
used, they may also have visibility to higher levels of the hierarchy
(especially if their allocations only make sense in the context of
higher levels of allocations) [#f5]_. There should be a principle of self
service here, that users in a project which is over quota should
always be able to figure out why that project is.

Service/Administrative users will have this read access for all projects.

This information will be fetched whenever a quota calculation is
needed. The service enforcing quotas should always assume it's calling
keystone to fetch the limit every time, even if this just turns into a
fast 304 HTTP NOT MODIFIED from keystone.

Items beyond scope
==================

During the Limits discussion at the Pike PTG, there was also interest
in another kind of system limitation, around rate limits that ensure a
healthy environment. The Swift team presented the perspective that
they were less interested in project level limits, but more in
limiting things to ensure the health of the cluster. Most of these
metrics included a time component (like iops).

While this is a very interesting question, and clearly a future need
around fairness and cluster health, when we talk about limits in the
context of this work we're only talking about fixed, integer amounts,
of resources.

Concrete path forward
=====================

The following is my best estimate on a path to move forward.

- Pike

  - Get general agreement on Keystone ownership of limits
  - Define keystone get / set limit interfaces (separate spec)
  - Create os-quotas lib with flat and strict account hierarchical
    limits
  - Implement Keystone get / set limit interfaces

- Queens

  - plumb unified limits support to one or more service projects (Nova
    stepping up). Start using by default, including API cutover (lots
    of bugs are going to fall out of this, we should consider it a
    whole cycle)

- Revelstoke

  - Convert rest of service projects to new limits model.
  - implement overbooking algorithm in os-quotas lib as experimental.


References
==========

These are existing sources of information out there around
hierarchical limits and quotas. They are included for reference only,
and are not meant to mean these will also be implemented.

- Existing Proposed Keystone Spec - https://review.openstack.org/#/c/363765/.

  This is a lower level specification (with POC code) that includes
  putting the limits information in the Token. Overuse of token was a
  primary concern. Tokens, are by definition, stale information, and
  can be long lived, thus an adminstrator changing limits would have
  no idea when the system would start enforcing them. Token bloat is
  also a concern for projects that have worker daemons and rely
  heavily on RPC, as it means more load on the RPC bus.

  This spec was presented at the Atlanta PTG, and is the spiritual
  basis for the above agreement.

- Proposed Nova Quotas Spec - https://review.openstack.org/#/c/429678.

  This is an overview spec that includes a mix of the rationale for
  putting limits in keystone. The use of quota classes. Some examples
  on overbooking. Elements of this spec have been carried into this
  unified spec. The Nova spec is considered defunct.

- Mailing list thread -
  http://lists.openstack.org/pipermail/openstack-dev/2017-March/113099.html

.. rubric:: Footnotes

.. [#f1] Nova has had a number of long standing bugs about mistyped
         project_ids ending up causing operator issues. The following
         spec was a partial fix for this -
         https://specs.openstack.org/openstack/nova-specs/specs/pike/approved/validate-project-with-keystone.html

.. [#f2] Long standing bug on this point
         https://bugs.launchpad.net/keystone/+bug/967832

.. [#f3] Nova has postponed even discussing hierarchical quotas fully
         in tree until both project validation
         (https://specs.openstack.org/openstack/nova-specs/specs/pike/approved/validate-project-with-keystone.html)
         and cells support
         (https://specs.openstack.org/openstack/nova-specs/specs/pike/approved/cells-count-resources-to-check-quota-in-api.html)
         are addressed. This would help remove much of the complexity.

.. [#f4] Known projects that check new limits don't exceed allocations: Nova

.. [#f5] Early modeling of Quota algorithms here -
         https://review.openstack.org/#/c/441203 - viewing the HTML
         rendered block diag content is the easiest way to understand this.

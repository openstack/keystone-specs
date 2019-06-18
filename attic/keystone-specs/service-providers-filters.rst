..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Expand endpoint filters to Service Providers
============================================

`bp service-provider-filters
<https://blueprints.launchpad.net/keystone/+spec/service-provider-filters>`_

This spec proposes to expand the endpoint filters feature to service providers.

Problem Description
===================

As of today, each authenticated user will get a `service catalog` as well as
list of trusted `service providers` within token response. This may pose
performance, security usability and last but not least, legal risks.  A client
scoping his token to a project or domain should be given a list of `service
providers` to whom he can utilize (given his current scope).
Getting a full list of cloud-wide trusted service providers will be very
inefficient in big cloud deployments, as huge lists of services providers will
be returned to all the users.  Public cloud providers should also limit service
providers in order not to reveal all partnered clouds (this could have some
legal implications). Lastly, token responses should only include service
providers allowed within current project/domain scope.

Proposed Change
===============

In order to scope a service provider to domains/projects, we need to expand the
`endpoint group filtering
<https://github.com/openstack/keystone-specs/blob/master/specs/juno/endpoint-group-filter.rst>`_
feature to support the filtering of service providers. Endpoint filtering
should allow for filtering service providers per scoped projects. This
limitation comes from the fact that endpoint filtering is available for
projects only. Effectively all service providers in the token response should
be filtered by configured filters.
The change would allow for:

1) Associating single service provider with a project
2) Setting (and later associating with a project) a ``service_providers_group``
   - group of service providers.

Additionally a configuration option would be added
``list_all_service_providers`` with valid options set to ``True`` or ``False``.
If the option would be ``True`` (and this would be default value) keystone
would return all service providers in the token response. This would also make
service providers filtering disabled. If the option would be ``False``, it
would list service providers that were filtered by a service providers
filtering.  It's also wise to expect, that after some sane deprecation time
``list_all_service_providers`` would become ``False`` as a default value.

Alternatives
------------

Propose completely new API, independent from OS-EP-FILTER API being
alternative for filtering service providers. The advantage of it would be the
ability to add more suitable options from the first iteration of this effort.
Additional filtering capabilities would be:
* associating a project with service provider(s)
* associating all projects within a domain with service provider(s)

Data Model Impact
-----------------

* None, the ``service_providers_group`` model should be able to handle
  ``service_providers`` as exemplified below::

    {
        "service_providers_group": {
            "description": "Example Service Provider Endpoint Group",
            "name": "SP-GROUP-1"
            "service_providers": [
                "sp1",
                "sp2",
                "sp3"
            ],
        }
    }

* The rule above should filter the ``service_provider`` with ID ``sp1`` and
  ``sp2`` and ``sp3``.

REST API Impact
---------------

It would be an extension to existing OS-EP-FILTER API, however new routes would
need to be added as we would handle additional use case.

Other End User Impact
---------------------

* The scope of the token will be used in order to return the list of service
  providers as described above.


Implementation
==============

Assignee(s)
-----------

* Marek Denis (marek-denis)
* Rodrigo Duarte (rodrigodsousa)

Work Items
----------

* Implement the handling of such filters in the endpoint filters backend and
  controller.
* Extend OS-EP-FILTER with associating resource (project) domain with a filter
* Reflect changes in the clients (`python-keystoneclient` and
  `python-openstackclient`) if necessary.

Documentation Impact
====================

* Identity v3 API OS-EP-FILTER should be updated to exemplify the
  service providers filtering.

References
==========

* https://blueprints.launchpad.net/keystone/+spec/endpoint-filtering
* https://blueprints.launchpad.net/keystone/+spec/multi-attribute-endpoint-grouping

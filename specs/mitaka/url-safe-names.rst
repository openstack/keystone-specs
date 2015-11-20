..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Optionally enforce URL-safe domain and project names
====================================================

`bp url-safe-naming <https://blueprints.launchpad.net/keystone/+spec/url-safe-naming>`_


In preparation for future support for hierarchical naming of projects and
domains, deprecate non url-safe names as well as optionally enforce
url-safe names.

Problem Description
===================

Once upon a time, in a release far away, there were just tenants in a flat
namespace. It was natural that tenant names had to be unique keystone-wide.
Subsequently we have added domains (to hold tenants, now called projects),
which meant that project names had only to be unique within that domain. We
now support hierarchies of projects (within a domain) and potentially may
support hierarchies of domains above projects. However, our name uniqueness
rules have not changed, so a project name must (effectively) be unique within
its hierarchy, and domain names must still be unique keystone-wide. One main
reason our rules have not changed is that we support scoping to domain and
project by name.

Given this new hierarchy support, our naming rules will become increasingly
onerous on customers. Not only for customer with deep project hierarchies,
but also will complicate any support for domain hierarchies for resellers.
In the future it would be ideal if you could address a node in a hierarchy with
a url-style addressing scheme, for example:

::

    {
        "auth": {
            "identity": {
                ...
            },
            "scope": {
                "project": {
                    "domain": {
                        "name": "acme.com"
                    },
                    "name": "development/sas/myproject"
                }
            }
        }
    }

The same addressing scheme could also be used for domains, if we require
support for domain hierarchies.

However, our current project and domain names are not url-safe, so we could not
guarantee to support any particular hierarchical naming scheme without risk of
breaking access to strangely named projects/domains.

Proposed Change
===============

It is proposed that we lay the groundwork to move to url-safe naming for all
projects and domains. A url-safe name is defined as one that only contains
`unreserved` characters as defined in section 2.3 of
`rfc3986 <http://tools.ietf.org/html/rfc3986>_`.

In particular, we would:

- Deprecate url-unsafe naming, so deprecation warnings would be logged for
  unsafe names.
- Provide a keystone-manage option to list the ID and name of any url-unsafe
  projects or domains, so that such names can subsequently be updated using the
  regular Identity API.
- Provide two configuration options (one for projects and one for domains) to
  enable enforcement.  These would accept three values: `off`, `new`, `strict`.
  These would be `off` by default, which means no change in behavior.
- If set to `new` then a request to create a new domain or project
  with a name that does not match the url safe criteria will be
  rejected.
- If the configuration option is set to `strict`, in addition to the
  restrictions defined for `new`, any existing unsafely-named domains or
  projects will be treated as disabled for the purposes of issuing a token
  scoped by project/domain name. Renaming the project or domain will re-enable
  it.


Future keystone releases would harden the above, moving towards a situation
when the configuration options are eventually deprecated and url-safe naming
is permanently enabled.

New features we release, for example domain hierarchies for resellers, could
insist that the appropriate url-safe configuration option was enabled before
that feature was supported.

To be clear, this proposal does not enable the actual use of hierarchical
naming, rather it lays the groundwork for us to do so in the future.

Alternatives
------------

None

Data Model Impact
-----------------

None

REST API Impact
---------------

None

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

None

Other Deployer Impact
---------------------

None, by default, although would require them to potentially rename projects
or domains if they want to start enforcing url-safe naming.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
    henry-nash

Work Items
----------

- Add support for new configuration options
- Add deprecation logging
- Add checking in create/update domain/project
- Add checking in auth to prevent scoping to unsafe entities
- Add support to keystone-manage for listing unsafely-named domains/projects

Dependencies
============

None

Testing
=======

None, above and beyond unit testing

Documentation Impact
====================

Changes to the Identity API to clarify naming.

References
==========

None

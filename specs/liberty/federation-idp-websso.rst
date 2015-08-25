..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Identity Provider Specific WebSSO
=================================

`bp federation-idp-websso <https://blueprints.launchpad.net/keystone/+spec/federation-idp-websso>`_

Currently the way keystone handles websso is global for each protocol in the
keystone instance. This requires some form of web page discovery service
implemented on top of keystone to allow users to select their provider.

In more simplistic deployments I want to be able to configure multiple saml
providers that can be selected from the horizon drop down menu rather than an
additional service.

Problem Description
===================

Currently WebSSO is configured in a global sense for keystone. You have to pick
a handler for /v3/auth/OS-FEDERATION/websso with a single apache module, this
route should offer some form of handler that will typically involve IdP
discovery and selection that redirects to the actual IdP handler.

To handle multiple IdPs this relies on the apache module having discovery
capability and the deployer constructing a webpage that shows the available
alternatives.

If your list of IdPs is fairly static it would be much better to simply have
horizon handle this. It already has a drop down selection to enable you to
select identity provider and it would be useful to be able to configure
individual identity providers to show up there. I should be able to select
google or facebook as my IdP and not care that they both use SAML from a
keystone perspective.

Proposed Change
===============

In addition to the /v3/auth/OS-FEDERATION/websso/{protocol} path we add a
similar handler at
/v3/auth/OS-FEDERATION/identity_providers/{idp_ip}/protocol/{protocol_id}/websso.
Rather than being dynamic, using remote-id to lookup the IdP that this relates
to this route will be handled only by the IdP specified in the URL.

This would allow the configuration of
/v3/auth/OS-FEDERATION/identity_providers/{idp_ip}/protocol/{protocol_id} that
is done for the CLI to also handle the websso case.

Alternatives
------------

We could continue with the global approach.

Security Impact
---------------

This is more secure than the current approach. When websso requests return to
keystone the global handler asserts that some configured Apache module has
verified the request. If I have both Google and Facebook SAML providers I must
validate against either of them and then we use the remote-ids parameter of a
keystone IdP to ensure that the URL that issued the assertion is one of those
already known and we discover mappings that way.

With this change we would have seperate URLs for instigating websso against
Google or Facebook and the Apache module is configured to only allow assertions
from the configure remote IdP.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This would be configured in `django_openstack_auth`. It would not be directly
used by a user.

Performance Impact
------------------

None

Other Deployer Impact
---------------------

This may change the way you configure apache for keystone. In the event that
you are also enabling federation via the command line it will simplify it as
you only need to configure the one location rather than one for CLI and one for
websso.

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the driver API, discussion of how
  other backends would implement the feature is required.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jamielennox

Other contributors:
  marekd
  lbragstad

Work Items
----------

* Add a
  /v3/auth/OS-FEDERATION/identity_providers/{idp_ip}/protocol/{protocol_id}/websso
  path to keystone.
* Allow configuring `django_openstack_auth` to use an individual IdP websso
  mechanism rather than the global endpoint.

Dependencies
============

None

Documentation Impact
====================

Document new URIs.

References
==========

`Bug report context <https://bugs.launchpad.net/bugs/1472060>`_

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Improved OpenID Connect Support
===============================

`bp improved-oidc-support <https://blueprints.launchpad.net/keystone/+spec/improved-oidc-support>`_


User access based on OpenID Connect is supported in keystone by leveraging the
Apache ``mod_auth_openidc`` module and the keystone federation APIs.

This involves setting the Apache server as an OpenID Connect client (Relying
Party) that will perform the configured authentication flow, getting the user
information (i.e. claims) from the standard OpenID Connect userinfo endpoint.
These extra claims include information such as email address, preferred
username, name, surname, etc. However, when using the OpenStack CLI, the oidc
RP is the CLI itself. The CLI will obtain an ``access_token`` from the OpenID
Connect Provider, and this token will be exchanged with an Oauth 2.0 protected
URL (previously configured by the keystone operator to do token instrospection
or local validation using ``mod_auth_openidc`` as well). In this case, only the
claims contained in the access token or returned by the introspection endpoint
will be present, as the userinfo endpoint is specific to OpenID Connect.

The above situation makes it difficult to implement complex policies that rely
on the information returned by the userinfo endpoint (such as email address)
and it presents a lack of behaviour consistency in the keystone setup. This
blueprint aims at fixing this issue, by adding an additional user information
retrieval for the OpenID Connect plugin.

Problem Description
===================

Currently OpenID Connect Provider (OP) as an external Identity Provider (IdP)
is supported by using:

* Apache + ``mod_auth_openidc`` configured as an OpenID Connect Relying Party
  (RP).

* Keystone with the Federation drivers enabled, using the
  ``keystone.auth.plugins.mapped.Mapped`` auth plugin.

According to `OpenID Connect specification`_, the Relying Party should be the
Oauth 2.0 client application that will contact the OP in order to get the
access/id tokens and eventually the additional user info from the corresponding
endpoint. The `Oauth 2.0 specification`_ states that a client is an application
making protected resource requests on behalf of the resource owner, and with
its authorization, not making assumptions on where it is being executed.

.. _OpenID Connect specification: https://openid.net/specs/openid-connect-core-1_0.html
.. _Oauth 2.0 specification: https://tools.ietf.org/html/rfc6749#section-1.1

In the dashboard case mentioned above, the OpenID RP is the Apache server,
therefore Apache is configured with the OpenID Connect client id and secret
that will be used for any of the OP grant types supported. Therefore, the
keystone administrator would register an OpenID Client in the OP, and add
its client id/secret to the ``mod_auth_openidc`` configuration. In this case,
since everything is handled within Apache and ``mod_auth_openidc``, Keystone
receives the access_token, id_token and all the additional grants obtained
from the userinfo endpoint in the HTTPD environment variables. The user
does not need to do anything with the OP, apart from the usual confirmation
that she is authenticating against the RP.

However, if the OpenStack CLIs are being used the RP is not the Apache server,
but the CLI (actually, ``keystoneauth1``). In this case, the user has to feed
the client id and secret to ``keystoneauth1``, therefore the user has to go to
the OP and create a new OpenID Connect client, fetch the discovery document
endpoint, client id and client secret and pass all to the library. Then through
keystoneauth performing the authentication flow using the requested grant type
against the OP, eventually obtaining an ``access_token``. This access_token is
then exchanged with an ``oauth20`` protected URL, that needs to be configured
to do token introspection against the RP, as in this `configuration guide`_.
Since this endpoint is an OAuth 2.0 endpoint it is not able to fetch any
additional claims from the userinfo endpoint, as this is something specific to
OpenID Connect.

.. _configuration guide: https://developer.ibm.com/opentech/2015/06/17/use-websphere-liberty-as-an-openid-connect-provider-for-openstack

Therefore, the keystone server does not have any additional claims obtained
from the userinfo endpoint apart from the ones that are already present in the
token, so it is not possible to create any mapping based on this (for example
group membership, email address, and so on). The tokens may include additional
claims, but this is not mandatory in the standard, being dependant on the OP
implementation. For example, Google's OAuth 2.0 introspection endpoint returns
these additional claims.

Following the OpenID Connect terminology, the RP should be keystone, and not
the user client. If so, when a user wants to authenticate with OpenID Connect
as an IdP the client should contact the federation URL that should be protected
with OpenID Connect. Then the authentication flow should be the same as in the
horizon+websso case, all handled by ``mod_auth_openidc`` and the keystone app
will get all the OpenID Connect claims (i.e from the id token and userinfo).
This way keystoneauth should not implement any openid logic apart from
intercepting the redirect request to the login endpoint and popping out a
browser (as in [2]).

[2] https://review.openstack.org/#/c/330006/

However, there are several disadvantages in doing this:

* Only one grant type can be configured per provider, therefore if a grant type
  of authz code is configured in the keystone server (the RP) the user won't be
  able to use the client credentials grant, even if the OP allows to do so

* All the code in keystoneauth regarding OpenID connect (that has been
  released) becomes useless and should be deprecated, as it should not handle
  any oidc grant type anymore.

* If the configuread grant type is the Authorization Code, the specification
  states that interaction with the user agent (e.g. a browser) is needed,
  therefore this cannot be used (per design) in a service library.

But it has a big advantage:

* The user does not need to create and manage OpenID Clients in the OP
  (thus it is not needed to handle the client id, secrets, etc.).

Nevertheless, CLI users may be expecting a similar experience to the one
obtained in other cloud providers (like Google Cloud Engine) where the
behaviour is like the one we have in place right now (i.e. the user needs to
create an OpenID Connect client and user the obtained client id and secret).

However, if we continue with this design, we can leave everything as it is
right now, but we need a specific OpenID Connect plugin in keystone that is
able to fetch the additional claims from the userinfo endpoint when it only
receives an id token. This way keystone will get all these additional claims
and the mapping set by the administrator can be based on them. If we do so,
operators should configure this plugin, instead of the current mapped plugin
(``keystone.auth.plugins.mapped.Mapped``).

Proposed Change
===============

The proposed change is to implement a specific and native OpenID Connect plugin
for Keystone. When this plugin is used, it will handle all the OpenID
Connection actions and flows, obtaining all the user claims. Afterwards the
plugin will continue as the vanilla mapped plugin, but the additional claims
will be present.

Alternatives
------------

The other alternative would be that all the OpenID Connect flow is done by
the Apache server where Keystone is running.  The advantages and disadvantages
of this are described in the "Problem Description" section.

Security Impact
---------------

This code will be managing the negotiation between keystone and the OpenID
Connect provider. However, there are Python modules (like `pyoidc`_) that are
`OpenID Connect certified implementations`_. We would implement only the logic
needed on top of these plugins.

.. _OpenID Connect certified implementations: https://openid.net/developers/certified/
.. _pyoidc: https://github.com/OpenIDC/pyoidc

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None, with the proposed solution.

Performance Impact
------------------

Additional calls need to be made to the external endpoints, that may introduce
a delay when responding to the user. This is already happening at the Apache
module.

Other Deployer Impact
---------------------

* There is an additional dependency on an external module (but this will remove
  a dependency on the Apache module).

* It would require additional configuration options and sections (one per
  provider).

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
* Alvaro Lopez (aloga)

Other contributors:
* None

Work Items
----------

1. Create an additional mapped plugin, implementing the described logic.

Dependencies
============

None.

Documentation Impact
====================

New documentation needs to be added on how to configure an OpenID Connect
provider.


References
==========

* `OpenID Connect Specifications <https://openid.net/developers/specs/>`
* `OAuth 2.0 specification <https://tools.ietf.org/html/rfc6749>`
* `Using Google OAuth 2.0 <https://developers.google.com/identity/protocols/OAuth2>`
* `Using Google OpenID Connect <https://developers.google.com/identity/protocols/OpenIDConnect>`
* `Using the Python client for GCE <https://cloud.google.com/compute/docs/tutorials/python-guide>`
* `OpenID Connect certified implementations <https://openid.net/developers/certified/>`
* `pyoidc <https://github.com/OpenIDC/pyoidc>`

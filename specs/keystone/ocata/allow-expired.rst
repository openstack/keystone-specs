..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Allow retrieving an expired token
=================================

`bp allow-expired <https://blueprints.launchpad.net/keystone/+spec/allow-expired>`_

Since the services split OpenStack has had a problem with long running
operations that reuse the user token timing out. An example of this is nova
initially validating a user token successfully, then it requests an image from
glance however in the mean time the token has expired and so glance rejects the
token. This causes very hard to debug timeouts and has resulted in services
doing operations as the service user instead of the user so that it can refresh
the token.

Problem Description
===================

OpenStack services only communicate with each other over the public REST APIs
that regular users have access to. A side effect of this is when you do service
to service communication it goes through the same token authentication process
regardless of whether the user has previously been validated by another
service.

Because keystone tokens are bearer tokens the easiest way to accomplish this
continuous user authentication was to have the service reuse the same token
when communicating with others. This works most of the time, however there is
an edge case that if the token expiry period is crossed between when the user
submits a token and the service passes the token on then token validation
will fail. At scale, this edge case becomes a serious problem and is a negative
experience for users because the operation may have been running a significant
time before failure occurs. It also presents a difficult rollback operation for
services because initial service to service requests may succeed which then
cannot be rolled back with the expired token.

We essentially need a way to have auth_token middleware give information for
expired tokens to the services in situations where the token has already been
validated by an existing trusted service.

A problem with this is because inter-service communication happens over public
APIs there is no way to know when a user's request entered a deployment. Glance
is unaware of whether a user wanted to list images or if nova is performing the
operation on behalf of a user.

Proposed Change
===============

One way we can determine that a token is being reused on behalf of a service is
for the service to submit its token along with the request. This is something
we already do in some scenarios for joint ownership of resources.

Keystone/auth_token middleware should support that if a token is submitted to
it along with an "X-Service-Token" with a service role (to distinguish it from
any old user trying to use an expired token) we should take that to mean the
token has been previously validated by a service we trust and accept the token
and fill the data expect by a service.

The keystone side of this is fairly simple. We add a flag ``?allow_expired=1``
to the existing ``GET /auth/tokens`` validation route that allows fetching an
expired token. The length of how long an expired token should be fetchable for
should be configurable on the server to prevent infinite reuse.

In ``auth_token`` middleware when a valid (must not be expired) service token
is presented with a request auth_token will make a validation request that may
return an expired token and fill this data for use by the service.

Then services who are making service to service requests using a user token
must also provide their current service token with a request to skip user token
expiry checking.

Alternatives
------------

Keystone vs auth_token middleware
+++++++++++++++++++++++++++++++++

There has been a few opinions on whether auth_token middleware should be
responsible for setting the ``allow_expired`` flag or whether both tokens
should be summited to keystone and the server be responsible for assessing the
validity of the service token.

In my opinion the addition of the ``allow_expired`` flag is not a security risk
so long as it is explicit opt-in behaviour. Thus so long as auth_token
middleware is only setting the flag in the appropriate place there is no
additional need to have keystone make this determination.

A downside to having this determination in auth_token middleware is that there
is no oslo.policy enforcement in auth_token middleware. It is therefore more
difficult to set rules on the service token that allow this behaviour (In
keystone it would simply be "validate_token: service_role:service" for allow
expired). We will therefore either need to add oslo.policy to auth_token
middleware or provide a simple ``service_token_roles`` config option for
validating the service token in middleware.

Others
++++++

Fixup trusts or OAuth to make them more generally usable.

Security Impact
---------------

This changes the security profile of OpenStack, it is actually to make
it more secure.  A token validator needs to opt in to this scheme, and thus
it cannot be used to unrevoke tokens etc.  The expection is that when one
service calls another, that call will be made using a service token in
conjunction with the users token.  Thus, it is the services identity
that is the link in the chain for authorization.  This is already the
case.

The most important benefit will be from the shortening of the expiry
times of tokens. Tokens used as a proxy for authentication should be
used within a few minutes of issuance. This change will allow that to happen.

Notifications Impact
--------------------

When a token is validated, it will need to annotate the use of this
flag, both presence and absence.

Other End User Impact
---------------------

Directly the change will be only in the validation API called by
Middleware. Indirectly, token expiration times can drop, and other
services will need to react accordingly. Horizon especially will have
to re-authenticate far more often than it does now.

Performance Impact
------------------

Interservice communication will now include a service token which is validated
seperately. At this time keystone does not support doing multiple token
validations at once so this is another validation request for each request.
This is a cacheable operation.

A positive outcome of this change would be for deployers to move to shorter
token expiry and there are performance implications of this. Two are:

* Shorter term tokens will mean more tokens are issued. However token reuse is
  not particularly good within OpenStack anyway and so at least for the
  immediate future usage patterns would probably not change.

* WebUI should be made aware of the shorter token timeouts and provide a means
  refresh. Since WebUIs do not cache the users credentials, there will likely
  be a push to explicitly extend the lifespan of unscoped tokens when fetched
  from Horizon or other specific web UIs, but that is beyond the scope of this
  spec. Alternatively Horizon could be provided a Service Token.

For completeness these are added however moving to a shorter default token
period is not the goal of this spec. For now we are primarily worried about the
long running operation case.

Other Deployer Impact
---------------------

Long term, each of the service policy files should indicate the roles
that a service token needs to have in order to make use of this
feature. Ideally, the roles will be more granular than just "service."

UUID tokens stored in the database will require holding on to expired
tokens in order to honor this change, which means providing a larger
window for token flush. PKI tokens that are not stored in the database
(or have been flushed) will require the full token body to honor this
change.

Developer Impact
----------------

When writing service to service communication developers will have to know to
pass an X-Service-Token.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Jamie Lennox <jamielennox@gmail.com>

Other contributors:
    Adam Young <ayoung@redhat.com>

Work Items
----------

* Add flag to Keystone server API

* Add flag to keystoneclient which performs the token validation step.

* Add step to keystonemiddleware that uses the above flags when a service token
  is present.

* Modify service to service communication to start passing a service token.

Dependencies
============

None

Documentation Impact
====================

Beyond the standard API documentation, this could be considered surprising
behaviour and a significant change. It should be well advertised and
documented.

References
==========

None yet.

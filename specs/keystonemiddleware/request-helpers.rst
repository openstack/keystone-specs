..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Request Helpers
===============

`bp request-helpers <https://blueprints.launchpad.net/keystonemiddleware/+spec/request-helpers>`_

With the addition of passing down an auth plugin from keystonemiddleware to the
services we are slowly starting to have some control over the way
authentication is used in service to service communication.

As part of this we can start to add helpers to this plugin that will
automatically and transparently make best practice decisions on behalf of
services.

Problem Description
===================

Service to Service communication is a mess. There is no standard way of
initiating calls and initiatives like Service tokens and Request-IDs are only
as successful as different services manually implement them.

Whilst ever these are implemented ad-hoc we will struggle to see full coverage
of these features.

Proposed Change
===============

The features are present in keystonemiddleware and keystoneclient to start
transparently adding information to Service to Service communications so that
we can handle these best practices and in future add any required additional
security information to these requests without having to modify every service.

We are already passing down an auth plugin to services. For clients that
support using an auth plugin (which is now most and the number is growing)
communicating with this service now simply involves taking the plugin from the
'keystone.token_auth' ENV variable auth_token middleware sets and initiating a
client with that plugin.

I would like to initially add X-Service-Token to the get_headers method so that
now whenever the plugin is used the Service Token (the token that is used to
authenticate auth_token middleware for validation) is added to requests that go
through this auth plugin. This is something that swift is doing manually today
and something that barbican and other services have expressed interest in to
allow them to control when a service creates a resource on behalf of a user.
Having this handled automatically would mean we could rely on something like
this for more advanced policy mechanisms.

This would also allow us to finally fix token binding as we would enforce a
token bind only upon a service token if present.

Additionally I would like to add the X-Openstack-Request-ID header. This header
is an attempt to allow better tracing of how a request flows through OpenStack.
By adding it to the auth plugin new services would gain support for this
request tracing with no additional effort.

Alternatives
------------

The alternative is to simply leave this information under the control of
individual services. New OpenStack wide initiatives will continue to be done
service by service.

Security Impact
---------------

We will see more distribution of the Service Token. This is not necessarily
inherent to this change but the concept of an X-Service-Token but as we are
going to add this header to many requests that currently don't require it this
will become more widely distributed.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

There is the additional cost of verifying X-Service-Tokens (regardless of token
format) that will be present when they otherwise would not have been, .

Other Deployer Impact
---------------------

None.


Developer Impact
----------------

There is ad-hoc support for these features already. We will want to remove them
from those services once we can assure that the new systems are being adopted.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  jamielennox

Other contributors:

Work Items
----------

* Add X-Service-Token support to the plugin
* Add X-OpenStack-Request-ID support to the plugin

The most notable work item is going to be the long term work of getting
services to use the provided auth plugin rather than the per-service context
management that is done today. This is a required and ongoing task regardless
of this blueprint.

Dependencies
============

None

Documentation Impact
====================

None

References
==========

* https://github.com/openstack/keystone-specs/blob/master/specs/keystonemiddleware/implemented/service-tokens.rst

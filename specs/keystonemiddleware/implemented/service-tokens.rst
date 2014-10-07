..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Service Token Composite Authorization
=====================================

`Proposed bp service-tokens
<https://blueprints.launchpad.net/keystonemiddleware/+spec/service-tokens>`_

The concept behind the service token is to provide a mechanism to allow
a service to handle a request in a different manner if the request is received
from the user directly or via another OpenStack service.

Simple Example Request Workflow::

    +----------------+
    |      User      |
    +-------+--------+
            | Access Image Data Request
            | X-AUTH-TOKEN: <end user token>
            | X-SERVICE-TOKEN: None
            |
    +-------v---------+
    |     Glance      |
    +-------+---------+
            | Access Image Data Request
            | X-AUTH-TOKEN: <original end user token>
            | X-SERVICE-TOKEN: <glance service user token>
            |
    +-------v---------+
    |      Swift      |
    +-----------------+


Problem Description
===================

In some cases it is desirable to handle a request differently depending on if
the request is made directly to the service instead of via an intermediary
service.

In the example workflow above, accessing the image data can only occur
via the Glance Service. If the user directly accesses the data via the
Swift API, the policy would enforce a ``HTTP 403 Forbidden``. This is to
ensure that the user cannot perform an update (maintain data integrity) of the
image without the ``Glance Service`` being aware of it.

The End User would make a request to ``Glance`` presenting the standard
Auth Token. The ``auth_token`` middleware running in ``Glance`` will
authorize the End User to make the API request for the image data.

``Glance`` will then make the request to ``Swift`` presenting both the ``End
User``'s token and the ``Glance Service User``'s token. The middleware running
in ``Swift`` will decode both the ``End User``'s token and the ``Glance Service
User``'s token and present the ``Swift`` service with the information from both
tokens. ``Swift`` will then make a policy decision, now capable of enforcing
that the request came from ``Glance`` instead of directly through the
Swift API.

Proposed Change
===============

The Keystone ``auth_token`` middleware will be modified to accept a second
token header: ``X-SERVICE-TOKEN``. If presented with the ``X-SERVICE-TOKEN``
header, it will decode the data from the service token and present it to the
underlying service in the same manner that the data from ``X-AUTH-TOKEN`` is
presented. The new data decoded from the ``X-SERVICE-TOKEN`` will use
separate and distinct naming indicating it originated from the
``X-SERVICE-TOKEN`` (e.g. if ``HTTP_X_ROLES`` was extracted from the
``X-AUTH-TOKEN`` header, the service equivalent will be
``HTTP_X_SERVICE_ROLES``).

This will allow the policy engine running inside the service to make policy
decisions on the data provided from both tokens and/or respond differently
based upon the presence of (or lack of presence of) either token.

Alternatives
------------

* Extend the Trust system to support more specific delegation of roles.

  Extending the Trust system and delegation capabilities would provide a
  significantly more difficult user experience for the end user. It would
  require delegating a Role to the service user explicitly and then making
  the request. The Service would need to know about the explicit Trust and
  know to scope a token to that Trust.

  This does not resolve the desire to secure the data from CRUD operations
  circumventing the service (e.g. ``Glance`` storing images in ``Swift``).

* Continue to simply utilize the user's permission and bearer token
  exclusively.

  This does not resolve the desire to secure the data from CRUD operations
  circumventing the service (e.g. ``Glance`` storing images in ``Swift``).

* A composite token requested from Keystone, which combines both original
  tokens into a new token.

  The original concept for this specification included requesting a new token
  from Keystone. This new token would contain elements from both of the
  original Tokens. This mechanism would provide the same benefits as the
  Service Token does and would allow the services to be aware of the entire
  path the request has taken (e.g. User -> Nova -> Glance -> Swift).

  The biggest downside would be needing to ask Keystone for a new token each
  step of the way (and forcing Keystone to resolve if it was allowed to issue
  a new Token). The extra round trip to Keystone would introduce significant
  overhead for minimal benefits: requiring each step to talk directly to
  Keystone eliminates the benefit of decoding/validating PKI tokens locally
  in the ``auth_token`` middleware. Complex policy decisions based upon the
  entire request path would be an extreme edge case and does not justify the
  overhead of extra round-trips to the Keystone service.


* OAUTH

  The current implementation of OAuth within Keystone has similar issues to
  expanding the Trust system. It also has fixed renewal requirements on a
  globally configured option. This would require continual re-authentication
  of the access (and/or request) tokens (end user intervention). Modifying
  the OAuth system to conform to the new use-cases would change the way
  users currently interact (and expect OAuth) to function, potentially
  breaking the contract defined in the OAuth APIs and requiring end users
  to update all tools/scripts/etc that currently utilize the OAuth system.

Data Model Impact
-----------------

None


REST API Impact
---------------

None


Security Impact
---------------

Describe any potential security impact on the system. Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

  This introduces support for a second token to transmit data for policy
  enforcement purposes to the services running behind ``auth_token``
  middleware.

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

  Policy enforcement will now have access to the data from the service
  token. This will not change any access to sensitive information without
  an explicit change to the policy for the service behind ``auth_token``
  middleware.

* Does this change involve cryptography or hashing?

  No

* Does this change require the use of sudo or any elevated privileges?

  No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

  No

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

  No more so than normal PKI tokens.


Notifications Impact
--------------------

No extra notification events will occur. CADF authentication events will be
emitted when service users authenticate to get their service token.

Other End User Impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* The session object in ``keystoneclient`` will need to support being able to
  send the ``X-SERVICE-TOKEN`` header. This should only impact service usage
  of the various python client libraries consuming ``keystoneclient`` session
  for authentication purposes.

Performance Impact
------------------

* Decoding the information from ``X-SERVICE-TOKEN`` header requires extra calls
  to CMS (subprocess) or request to ``Keystone`` in the case of UUID tokens.

* A service should attempt to re-use its service token as long as the token
  is not about to expire. This will limit round-trips to Keystone to request a
  new token to provide in the ``X-SERVICE-TOKEN`` header. Each service will be
  responsible for refreshing its service token as needed.

* The request overhead due to the included token(s) doubles.

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

  Any service updated to leverage sending a ``X-SERVICE-TOKEN`` header will
  need to have options for service user credentials added.

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

  Any service leveraging the ``X-SERVICE-TOKEN`` header will need policy
  explicitly built to enforce based upon the data in the extra header. Any
  service that wishes to send the ``X-SERVICE-TOKEN`` header will need
  to be configured with service user credentials.

* Please state anything that those doing continuous deployment, or those
  upgrading from the previous release, need to be aware of. Also describe
  any plans to deprecate configuration values or features. For example, if we
  change the directory name that instances are stored in, how do we handle
  instance directories created before the change landed? Do we move them? Do
  we have a special case in the code? Do we assume that the operator will
  recreate all the instances in their cloud?

  The changes to the services leveraging the composite authorization
  implementation should be (mostly) transparent. It will be a policy deployment
  to enable (at most).

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* Developers will need to leverage the new ``Session`` object from
  ``keystoneclient`` for authentication to external services to ensure the
  ``X-SERVICE-TOKEN`` is properly populated when making requests to
  external OpenStack services.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Morgan Fainberg (mdrnstm)

Other contributors:
  Stuart McLaren

Work Items
----------

* ``auth_token`` middleware support for decoding and presenting data from
  ``X-SERVICE-TOKEN``

* KeystoneClient session object support for ``X-SERVICE-TOKEN``

* Cross Project collaboration to leverage and/or document policy
  recommendations that use the data from ``X-SERVICE-TOKEN`` for enforcement.


Dependencies
============

None

Testing
=======

Unit tests for ``auth_token`` middleware will need to be developed to validate
data from both ``X-AUTH-TOKEN`` and ``X-SERVICE-TOKEN`` are being presented
to the underlying service.

As services are capable of emitting new header and enforcing policy on the
extra information received through the ``auth_token`` middleware, Tempest
tests will be needed to validate the new policy models. Consuming the data
from the new ``X-SERVICE-TOKEN`` header will use the logical ``and`` and
logical ``or`` mechanisms within the policy DSL to enforce on the combination
of service-token data and user-token data.


Documentation Impact
====================

* Deployment doc change and best practices (service user creation, role
  assignment, etc) will need to be written.

* Example policy files will need to be created that show how to use the new
  data provided from ``X-SERVICE-TOKEN`` when making policy enforcement
  decisions.


References
==========

* Previous Example implementation: https://review.openstack.org/#/c/88366/

* http://dolphm.com/openstack-juno-design-summit-outcomes-for-keystone/#compositetokens

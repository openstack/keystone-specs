..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Keystone to Keystone federation
===============================

`bp keystone-to-keystone-federation
<https://blueprints.launchpad.net/keystone/+spec/keystone-to-keystone-federation>`_

Enable the ability to federate identities between Keystone instances, with
Keystone acting on behalf of the user to deliver cross-cloud authorization and
a homogeneous cross-cloud service list.

Problem Description
===================

Actors
------

* *Cloud Implementer*: The person or group of people deploying and configuring
  an OpenStack cloud.

* *Service Provider (SP)*: An entity that offers an OpenStack compatible cloud
  service (public or private).

* *Identity Provider (IdP)*: The federated identity provider of the end user.
  The service responsible for managing the authentication of the end user.

* *End user*: An identity with access to one or many OpenStack clouds.

In the Icehouse release, we identified a way to establish trust between an IdP
and Keystone. This allowed Cloud Implementers to set-up single-sign-on when
configuring their cloud and not incur the administrative overhead that comes
with provisioning identities in Keystone. Keystone trusts the IdP enough to
**accept federated identities from** them. However, there is no way right now
for one Keystone to trust another Keystone. This is where one Keystone trusts
another to **assert federation identities to** it.

Use Case 1: Multiple Re-sellers
-------------------------------

CERN has one OpenStack cloud set-up within their data center. They have
Keystone identity backed by an LDAP instance. They would like to allocate cloud
workload between multiple public service providers, should their cloud not have
sufficient resources.

Use Case 2: Cloud bursting
--------------------------

ACME has one OpenStack cloud set-up within their data center that surfaces a
large eCommerce site. They have a promotion coming up on Black Friday, but do
not have the infrastructure in place to handle the projected traffic. They are
even unsure of what the projected traffic could be. They would like to
auto-scale their application across SPs.

Use Case 3: Multiple in-house OpenStacks
----------------------------------------

ACME has several OpenStack clouds, and each one maintains its own set of
assignments. They would like to enable identity federation across their
multiple clouds so that an identity accessing each cloud doesn't need to
maintain several tokens and credentials.

Use Case 4: Storage-only Provisioning
-------------------------------------

A service provider wishes to provide only storage services to external users.
Hence, in any federation, the SP must be able to "advertise" only it's Swift
service endpoints, and no others. Furthermore, the storage provider may wish to
enforce access policies (who can access which containers) based on the set of
roles that it defines. Hence, the User's Keystone must be able to acquire this
set of roles and assign them to users accordingly.

Requirements
------------

1. The solution needs to be recursive, so that if Cloud A burst to Cloud B when
   it is overloaded, and Cloud B burst to Cloud C when it is overloaded, then
   jobs from A can be burst to C via B.

2. The solution needs to require minimum changes to existing non-Keystone
   components, i.e., both clients and OpenStack services, or to put it another
   way, the majority of the changes should be applied to Keystone.

3. (A) If changes have to be applied to either clients or OpenStack services,
   then changes to clients should take precedence over changes to OpenStack
   services. It would be optimum if only the Keystone client needs to change in
   this situation.

   OR

   (B) The current user-experience, that is used by non-federation-aware
   clients, should not need to change. Thus changes to service should take
   precedence over changes to clients.

4. The solution must be as secure as the existing implementation of Keystone
   and Openstack. Specifically, we must not degrade the overall security by
   the introduction of Keystone federation. On the other hand it does not need
   to be more secure since attackers will always go for the weakest link.

5. The solution must not reduce the functionality of existing clients and
   services.

6. The solution must work for both federated users and non-federated users.

7. The solution should not unduly impact on the non-functional aspects of
   existing OpenStack services e.g. performance, scalability etc.

8. The local Keystone administrator must be able to have full control over the
   external entities that he trusts. This is relatively easy to achieve for
   direct trust relationships. but is more complex for indirect trust
   relationships, e.g., Keystone A trusts IdP 1 and external Keystone B, but
   not IdP2. Keystone B trusts IdP 1 and IdP 2. Keystone A must be able to stop
   users from IdP2 authenticating to Keystone B and then accessing its services
   via this indirect route.

9. Should a token be revoked (either by its holder or issuer), there needs to
   be some way for all recipients (i.e. CSPs) to know about this event. E.g.,
   there is a revocation service that the recipient can contact, or the issuer
   broadcasts out to its trusted CSPs, so that the revocation is all
   encompassing. The alternative is for the IdP to issue short lived tokens
   that will never be revoked, e.g., as in SAML. Keystone should be able to
   handle both of these scenarios.

Proposed Change
===============

1. Establish a trust between the both entities. BETA, the trusted service
   provider is an external service (likely another Keystone), which trusts the
   local Keystone to **assert federated identities to** it.

   This will allow the cloud implementer to explicitly manage trust
   relationships. For example, BETA is a service provider. ACME would like to
   use BETA's resources. To establish a relationship between the two, ACME
   would trust BETA as a service provider, and BETA would trust ACME as an
   identity provider. That will allow ACME to burst into BETA cloud but not the
   other way around.

2. ACME's Keystone should only have knowledge of BETA's authentication URL.
   Once ACME's users have authenticated with ACME, they may transform their
   Keystone token into a SAML assertion, consumable by BETA.

   It is worth noting that this new capability is built on and leverages the
   existing Icehouse SAML federation capabilities. As such it does not require
   modifications to Keystone's existing ability to consume federation
   assertions. Instead it just adds the ability for Keystone to generate
   assertions in addition to consuming them.

3. BETA's Keystone, upon receiving a SAML assertion will need to be able to map
   ACME's user to a role and group in BETA.

::

            Figure 1: Creating a trust relationship

     +----------+                                 +---------------+
     |          |<--- (1) Add BETA as an SP   ----|               |
     |   ACME   |                                 |  ACME Cloud   |
     |          |                                 |  Implementer  |
     |          |<--- (2) Add IdP1 as an IdP  ----|               |
     +----------+                                 +---------------+
                                                      |
                                                      | Out of Band
                                                      |
     +----------+                                 +---------------+
     |          |<--- (3) Add ACME as an IdP  ----|               |
     |   BETA   |                                 |  BETA Cloud   |
     |          |                                 |  Implementer  |
     |          |<--- (4) Add DELTA as an SP  ----|               |
     +----------+                                 +---------------+
                                                      |
                                                      | Out of Band
                                                      |
     +----------+                                 +---------------+
     |          |<--- (5) Add BETA as an IDP  ----|               |
     |  DELTA   |                                 |  DELTA Cloud  |
     |          |                                 |  Implementer  |
     |          |                                 |               |
     +----------+                                 +---------------+

The flow illustrated in Figure 1 includes the following steps:

1. Cloud Implementer at ACME adds BETA as V3 Regions, supplying BETA's external
   authentication URL.

2. (optional) User's at ACME may either authenticate locally or through a
   trusted IdP.

3. Cloud Implementer at BETA adds ACME as a trusted IdP via existing
   OS-FEDERATION APIs. The Cloud Implementer must also create a Mapping and
   associate the Mapping with ACME IdP and a Protocol.

4. (optional) Cloud Implementor at BETA adds DELTA as a trusted SP etc.

5. (optional) Cloud Implementor at DELTA adds BETA as a trusted IdP etc.

::

            Figure 2: Authentication Flow

                                                    +----------+
                                                    |          |
          +-------(1') IdP sends assertion ---------|   IdP    |
          |                                         |          |
          |                                         +----------+
          |                                              |
          |                                             (1)
          |                                         Authenticates
          |                                              |
          v                                              |
     +----------+                                 +---------------+
     |          |                                 |               |
     |          |----(2)--- Catalog returned ---->|               |
     |   ACME   |                                 |     USER      |
     |          |<---(3)----  Scoped Token  ------|               |
     |          |                                 |               |
     |          |----(4)-- SAML token returned -->|               |
     +----------+                                 +---------------+
                                                      |      ^
                                                      |      |
                                                      |      |
                                                      |      |
     +---------+          User authenticates          |      |
     |         |<---(5)- with SAML data from (4) -----'      |
     |  BETA   |            OS-FEDERATION                    |
     |         |                                             |
     |         |----(6)--- BETA token returned --------------'
     +---------+

The flow illustrated in Figure 2 includes the following steps:

1. User authenticates with ACME, either locally or through their local IdP.

2. User is returned a service catalog with ACME's services as well as URLs of
   the identity services of trusted service providers.

3. User will send his scoped token, and "BETA" (a region ID) to ACME.

4. ACME cloud returns the a SAML assertion.

Resume previously established Icehouse federation behaviour:

5. User authenticates with BETA, presenting the SAML assertion returned from
   (4). Note that because ACME is acting as a trusted proxy to the user's IdP,
   the history of where the user actually authenticated (the user's IdP) may be
   lost to BETA, unless it is specifically identified in the SAML assertion.

6. BETA receives the SAML assertion and resumes the established federation
   authenticaion flow.

Alternatives
------------

1. Require the user to obtain multiple credentials in advance, one for each CSP
   he is going to visit, so that he can be authorised for any CSP service he
   wants to use by using the correct local token.

   This is very undesirable as it requires client configuration or code changes
   each time a new CSP is trusted, and it makes the client code more complex to
   handle multiple tokens with different scope and varying expiration times.

2. All CSP services will cater for users from any trusted IdP or proxy IdP by
   accepting every credential issued within the circle of trust, so regardless
   of the token the user has, he can obtain the service that is provided. (This
   requires every service such as Swift to be altered to accept any credential,
   not just locally issued ones).

   This is very undesirable as it means that every OpenStack service will need
   to change in order to support remote users

3. Each CSP hosts a local token exchange service (which could be Keystone) so
   that when users land at the CSP then whatever credential they currently hold
   can be exchanged for the local one used by the CSP's services.

   This is a workable model if Keystone becomes the local token exchange
   service. None of the CSP's services need to change. If the user hits
   Keystone first it will simply exchange the remote token for a local one and
   then redirect the user to the local services. If the user hits a service
   first, it can simply send every token it receives from every user to the
   local Keystone to validate and Keystone will send back to the service
   information that it can use and understand for authorisation purposes.

4. A new circle of trust (CoT) token service is set up which allows users to
   get "international" credentials prior to visiting remote CSPs. Users can
   present these CoT tokens to every CSP in the circle of trust. The CSP sends
   the CoT token to the CoT Service for validation, then gives the user a local
   credential valid for the CSP's services.

   This is also a workable solution. If Keystone gives users two tokens: one
   issued by itself that is valid for its local services, and one issued by the
   CoT service that is valid for all remote services in the CoT. When the user
   goes to a remote CSP, this now behaves as in iii) above, except that the CoT
   token is sent to the CoT service for validation rather than the local
   Keystone service, and it returns a token valid for the local CSP.

Security Impact
---------------

This change does touch several sensitive data components, specifically Tokens:

* If the SP (BETA) wishes to no longer accept requests from the IdP, the SP
  admin can simply disable the ACME IdP entry.

* If the IdP deletes the SP, then the SP should delete the IdP as well. This is
  an out-of-band operation, and beyond the scope of this specification.

* Possible information leakage. The token could contain IDs and roles, etc.
  from ACME and that information is available to BETA. Though the information
  may not be important at the moment, it could be important in the future.

* Does this increase the surface area of an attack if a cloud is compromised?
  For example, say ACME has a partnership with BETA and DELTA as SPs. A
  compromise of BETA is a Compromise of ACME and DELTA at the token level.

  * This will no longer be an issue since SAML assertions are X509 signed.

Notifications Impact
--------------------

* Keystone will emit a CADF notification (to authenticated and authorized
  identities), revocations should occur for previously issued assertions.

Other End User Impact
---------------------

``python-keystoneclient`` may need to handle multiple tokens (one token per
federated cloud).

Performance Impact
------------------

The following operations may impact performance:

* Since there could be as many service providers as regions, the catalog may
  become larger. However this limitation exist today a catalog can have many
  endpoints and also become pretty large with the endpoints that a user has
  access to.

* More certificates to validate tokens against.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* stevemar (Steve Martinelli <stevemar@ca.ibm.com>)

* marekd (Marek Denis <marek.denis@cern.ch>)

Other contributors:

* henrynash (Henry Nash <henryn@linux.vnet.ibm.com>)

Work Items
----------

* Modify Regions API to define an external authentication URL.

* Define and implement a method for storing the signing keys of the external
  IdP. This work may potentially be done using Barbican.

* Define and implement a method to transform a Keystone token into a SAML
  assertion.

* Define and implement a method of invalidating tokens across service
  providers.

Dependencies
============

None

Documentation Impact
====================

Extensive documentation will have to be performed to describe any new
configurations necessary.

References
==========

* `Blueprint
  <https://blueprints.launchpad.net/keystone/+spec/keystone-to-keystone-federation>`_

* `Drafting of specification
  <https://etherpad.openstack.org/p/service-provider-federation-spec>`_

* `Discussion from 6/9/2014
  <https://etherpad.openstack.org/p/Keystone-to-Keystone-federation-items>`_

* `Presentation from Atlanta 2014 session
  <https://www.openstack.org/assets/presentation-media/os-federation-final.pdf>`_

* `Federation design session notes
  <https://etherpad.openstack.org/p/juno-keystone-federation>`_

* `Trusting other Keystones
  <http://www.slideshare.net/davidwchadwick/keystone-trust>`_

* `Federation @ Atlanta Summit
  <http://dolphm.com/openstack-juno-design-summit-outcomes-for-keystone/#identityfederation>`_

* `Juno Hackathon details
  <https://etherpad.openstack.org/p/keystone-juno-hackathon>`_

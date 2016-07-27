====================================
OpenStack Identity API v2.0 overview
====================================

The OpenStack Identity API is implemented using a RESTful web service
interface. All requests to authenticate and operate against the
OpenStack Identity API should be performed using SSL over HTTP (HTTPS)
on TCP port 443.

OpenStack Identity enables clients to obtain tokens that permit access
OpenStack cloud services.

Intended audience
-----------------

This reference is for software developers who develop applications that
use the Identity API for authentication.

This reference assumes that the reader is familiar with RESTful web
services, HTTP/1.1, and JSON or XML serialization formats.

Identity concepts
-----------------

To use OpenStack Identity, you must be familiar with these key concepts:

**User**
    A digital representation of a person, system, or service that uses
    OpenStack cloud services. OpenStack Identity authentication services
    validate that an incoming request is being made by the user who
    claims to be making the call.

    Users have a login and may be assigned tokens to access resources.
    Users may be directly assigned to a particular tenant and behave as
    if they are contained in that tenant.

**Credentials**
    Data that belongs to, is owned by, and generally only known by a
    user that the user can present to prove their identity.

    Examples include:

    -  A matching username and password

    -  A matching username and API key

    -  A token that was issued to you

**Authentication**
    In the context OpenStack Identity, the act of confirming the
    identity of a user or the truth of a claim. OpenStack Identity
    confirms that an incoming request is being made by the user who
    claims to be making the call by validating a set of claims that the
    user is making.

    These claims are initially in the form of a set of credentials
    (username & password, or username and API key). After initial
    confirmation, OpenStack Identity issues the user a token, which the
    user can then provide to demonstrate that their identity has been
    authenticated when making subsequent requests.

**Token**
    An arbitrary bit of text that is used to access resources. Each
    token has a scope that describes which resources are accessible with
    it. A token may be revoked at anytime and is valid for a finite
    duration.

    While OpenStack Identity supports token-based authentication in this
    release, the intention is for it to support additional protocols in
    the future. The intent is for it to be an integration service
    foremost, and not aspire to be a full-fledged identity store and
    management solution.

**Tenant**
    A container used to group or isolate resources and/or identity
    objects. Depending on the service operator, a tenant can map to a
    customer, account, organization, or project.

**Service**
    An OpenStack service, such as Compute (Nova), Object Storage
    (Swift), or Image Service (Glance). A service provides one or more
    endpoints through which users can access resources and perform
    operations.

**Endpoint**
    A network-accessible address, usually described by a URL, where a
    service may be accessed. If using an extension for templates, you
    can create an endpoint template, which represents the templates of
    all the consumable services that are available across the regions.

**Role**
    A personality that a user assumes when performing a specific set of
    operations. A role includes a set of rights and privileges. A user
    assuming that role inherits those rights and privileges.

    In OpenStack Identity, a token that is issued to a user includes the
    list of roles that user can assume. Services that are being called
    by that user determine how they interpret the set of roles a user
    has and to which operations or resources each role grants access.

    It is up to individual services such as the Compute service and
    Image service to assign meaning to these roles. As far as the
    Identity service is concerned, a role is an arbitrary name assigned
    by the user.


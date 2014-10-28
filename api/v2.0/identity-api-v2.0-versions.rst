====================================
OpenStack Identity API v2.0 Versions
====================================

The OpenStack Identity API uses both a URI and a MIME type versioning
scheme. In the URI scheme, the first element of the path contains the
target version identifier (for example,
https://identity.api.openstack.org/ v2.0/). The MIME type versioning
scheme uses HTTP content negotiation where the ``Accept`` or
``Content-Type`` headers contains a MIME type that includes the version
ID as a parameter (application/vnd.openstack.identity+xml;version=1.1).
A version MIME type is always linked to a base MIME type
(application/xml or application/json). If conflicting versions are
specified using both an HTTP header and a URI, the URI takes precedence.

**Example: Request with MIME type versioning**

.. code::

    GET /tenants HTTP/1.1
    Host: identity.api.openstack.org
    Accept: application/vnd.openstack.identity+xml;version=1.1
    X-Auth-Token: eaaafd18-0fed-4b3a-81b4-663c99ec1cbb


**Example: Request with URI versioning**

.. code::

    GET /v1.1/tenants HTTP/1.1
    Host: identity.api.openstack.org
    Accept: application/xml
    X-Auth-Token: eaaafd18-0fed-4b3a-81b4-663c99ec1cbb


.. note::

    The MIME type versioning approach allows for the creation of permanent
    links, because the version scheme is not specified in the URI path:
    ``https://api.identity.openstack.org/tenants/12234``.

If a request is made without a version specified in the URI or through
HTTP headers, a multiple-choices response (300) provides links and MIME
types to available versions.

**Example: Multiple choices: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="utf-8"?>
    <choices xmlns="http://docs.openstack.org/common/api/v1.0" xmlns:atom="http://www.w3.org/2005/Atom">
        <version id="v1.0" status="DEPRECATED">
            <media-types>
                <media-type base="application/xml"
                    type="application/vnd.openstack.identity+xml;version=1.0"/>
                <media-type base="application/json"
                    type="application/vnd.openstack.identity+json;version=1.0"/>
            </media-types>
            <atom:link rel="self" href="http://identity.api.openstack.org/v1.0"/>
        </version>
        <version id="v1.1" status="CURRENT">
            <media-types>
                <media-type base="application/xml"
                    type="application/vnd.openstack.identity+xml;version=1.1"/>
                <media-type base="application/json"
                    type="application/vnd.openstack.identity+json;version=1.1"/>
            </media-types>
            <atom:link rel="self" href="http://identity.api.openstack.org/v1.1"/>
        </version>
        <version id="v2.0" status="BETA">
            <media-types>
                <media-type base="application/xml"
                    type="application/vnd.openstack.identity+xml;version=2.0"/>
                <media-type base="application/json"
                    type="application/vnd.openstack.identity+json;version=2.0"/>
            </media-types>
            <atom:link rel="self" href="http://identity.api.openstack.org/v2.0"/>
        </version>
    </choices>

**Example: Multiple choices: JSON response**

.. code:: javascript

    {
        "choices": [
            {
                "id": "v1.0",
                "status": "DEPRECATED",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.0"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=1.0"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=1.0"
                        }
                    ]
                }
            },
            {
                "id": "v1.1",
                "status": "CURRENT",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.1"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=1.1"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=1.1"
                        }
                    ]
                }
            },
            {
                "id": "v2.0",
                "status": "BETA",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v2.0"
                    }
                ],
                "media-types": {
                    "values": [
                        {
                            "base": "application/xml",
                            "type": "application/vnd.openstack.identity+xml;version=2.0"
                        },
                        {
                            "base": "application/json",
                            "type": "application/vnd.openstack.identity+json;version=2.0"
                        }
                    ]
                }
            }
        ],
        "choices_links": ""
    }


New features and functionality that do not break API-compatibility are
introduced in the current version of the API as extensions (see the
following section) and the URI and MIME types remain unchanged. Features
or functionality changes that would necessitate a break in
API-compatibility require a new version, which results in URI and MIME
type versions being updated accordingly. When new API versions are
released, older versions are marked as ``DEPRECATED``. Providers should
work with developers and partners to ensure adequate migration time to
the new version before deprecated versions are discontinued.

Your application can programmatically determine available API versions
by performing a **GET** on the root URL (such as, with the version and
everything to the right of it truncated) returned from the
authentication system. Note that an Atom representation of the versions
resources is supported when issuing a request with the ``Accept`` header
containing application/atom+xml or by adding a .atom to the request URI.
This enables standard Atom clients to track version changes.

**Example: List versions: HTTP request**

.. code::

    GET HTTP/1.1
    Host: identity.api.openstack.org



Normal response code(s):200, 203

Error response code(s): badRequest (400), identityFault (500),
serviceUnavailable(503)

This operation does not require a request body.

**Example: List versions: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>

    <versions xmlns="http://docs.openstack.org/common/api/v1.0"
              xmlns:atom="http://www.w3.org/2005/Atom">

      <version id="v1.0" status="DEPRECATED"
              updated="2009-10-09T11:30:00Z">
         <atom:link rel="self"
                    href="http://identity.api.openstack.org/v1.0/"/>
      </version>

      <version id="v1.1" status="CURRENT"
              updated="2010-12-12T18:30:02.25Z">
         <atom:link rel="self"
                    href="http://identity.api.openstack.org/v1.1/"/>
      </version>

      <version id="v2.0" status="BETA"
              updated="2011-05-27T20:22:02.25Z">
         <atom:link rel="self"
                    href="http://identity.api.openstack.org/v2.0/"/>
      </version>

    </versions>



**Example: List versions: JSON response**

.. code:: javascript

    {
        "versions": [
            {
                "id": "v1.0",
                "status": "DEPRECATED",
                "updated": "2009-10-09T11:30:00Z",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.0/"
                    }
                ]
            },
            {
                "id": "v1.1",
                "status": "CURRENT",
                "updated": "2010-12-12T18:30:02.25Z",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v1.1/"
                    }
                ]
            },
            {
                "id": "v2.0",
                "status": "BETA",
                "updated": "2011-05-27T20:22:02.25Z",
                "links": [
                    {
                        "rel": "self",
                        "href": "http://identity.api.openstack.org/v2.0/"
                    }
                ]
            }
        ],
        "versions_links": []
    }



You can also obtain additional information about a specific version by
performing a **GET** on the base version URL (for example,
https://identity.api.openstack.org/v2.0/). Version request URLs should
always end with a trailing slash (/). If the slash is omitted, the
server might respond with a 302 redirection request. Format extensions
might be placed after the slash (for example,
https://identity.api.openstack.org/v2.0/.xml). Note that this is a
special case that does not hold true for other API requests. In general,
requests such as /tenants.xml and /tenants/.xml are handled
equivalently.

**Example: Get version details: HTTP request**

.. code::

    GET HTTP/1.1
    Host: identity.api.openstack.org/v2.0/


Normal response code(s):200, 203

Error response code(s): badRequest (400), identityFault (500),
serviceUnavailable(503)

This operation does not require a request body.

**Example: Get version details: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <version xmlns="http://docs.openstack.org/identity/api/v2.0"
        status="stable" updated="2013-03-06T00:00:00Z" id="v2.0">
        <media-types>
            <media-type base="application/json"
                type="application/vnd.openstack.identity-v2.0+json"/>
            <media-type base="application/xml"
                type="application/vnd.openstack.identity-v2.0+xml"/>
        </media-types>
        <links>
            <link href="http://localhost:5000/v2.0/" rel="self"/>
            <link
                href="http://docs.openstack.org/api/openstack-identity-service/2.0/content/"
                type="text/html" rel="describedby"/>
            <link
                href="http://docs.openstack.org/api/openstack-identity-service/2.0/identity-dev-guide-2.0.pdf"
                type="application/pdf" rel="describedby"/>
        </links>
    </version>


**Example: Get version details: JSON response**

.. code:: javascript

    {
        "version": {
            "status": "stable",
            "updated": "2014-04-17T00:00:00Z",
            "media-types": [
                {
                    "base": "application/json",
                    "type": "application/vnd.openstack.identity-v2.0+json"
                },
                {
                    "base": "application/xml",
                    "type": "application/vnd.openstack.identity-v2.0+xml"
                }
            ],
            "id": "v2.0",
            "links": [
                {
                    "href": "http://23.253.228.211:5000/v2.0/",
                    "rel": "self"
                },
                {
                    "href": "http://docs.openstack.org/api/openstack-identity-service/2.0/content/",
                    "type": "text/html",
                    "rel": "describedby"
                },
                {
                    "href": "http://docs.openstack.org/api/openstack-identity-service/2.0/identity-dev-guide-2.0.pdf",
                    "type": "application/pdf",
                    "rel": "describedby"
                }
            ]
        }
    }


.. annegentle: Removed paragraph and note about machine readable link and WADL
    because there's nothing machine readable on docs.openstack.org/api/ after we
    get these specs here. Need to investigate this -- is it sufficient to
    redirect:
    http://docs.openstack.org/api/openstack-identity-service/2.0/content/
    to
    http://specs.openstack.org/?

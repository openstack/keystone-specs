======================================
OpenStack Identity API v2.0 Extensions
======================================

The OpenStack Identity API is extensible. Extensions serve two purposes:
They allow the introduction of new features in the API without requiring
a version change and they allow the introduction of vendor specific
niche functionality. Applications can programmatically determine what
extensions are available by performing a **GET** on the /extensions URI.
Note that this is a versioned request - that is, an extension available
in one API version might not be available in another.

=======  ===========  ======================================
Verb     URI          Description
**GET**  /extensions  Returns a list of available extensions
=======  ===========  ======================================

Normal response code(s):200, 203

Error response code(s): badRequest (400), identityFault (500),
serviceUnavailable(503)

This operation does not require a request body.

Each extension is identified by two unique identifiers, a namespace and
an alias. Additionally an extension contains documentation links in
various formats.

**Example: List extensions: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <extensions xmlns="http://docs.openstack.org/common/api/v1.0"
                xmlns:atom="http://www.w3.org/2005/Atom"/>


**Example: List extensions: JSON response**

.. code:: javascript

    {
        "extensions": {
            "values": [
                {
                    "updated": "2013-07-07T12:00:0-00:00",
                    "name": "OpenStack S3 API",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/s3tokens/v1.0",
                    "alias": "s3tokens",
                    "description": "OpenStack S3 API."
                },
                {
                    "updated": "2013-07-23T12:00:0-00:00",
                    "name": "OpenStack Keystone Endpoint Filter API",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api/blob/master/openstack-identity-api/v3/src/markdown/identity-api-v3-os-ep-filter-ext.md",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/OS-EP-FILTER/v1.0",
                    "alias": "OS-EP-FILTER",
                    "description": "OpenStack Keystone Endpoint Filter API."
                },
                {
                    "updated": "2013-12-17T12:00:0-00:00",
                    "name": "OpenStack Federation APIs",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/OS-FEDERATION/v1.0",
                    "alias": "OS-FEDERATION",
                    "description": "OpenStack Identity Providers Mechanism."
                },
                {
                    "updated": "2013-07-11T17:14:00-00:00",
                    "name": "OpenStack Keystone Admin",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/OS-KSADM/v1.0",
                    "alias": "OS-KSADM",
                    "description": "OpenStack extensions to Keystone v2.0 API enabling Administrative Operations."
                },
                {
                    "updated": "2014-01-20T12:00:0-00:00",
                    "name": "OpenStack Simple Certificate API",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/OS-SIMPLE-CERT/v1.0",
                    "alias": "OS-SIMPLE-CERT",
                    "description": "OpenStack simple certificate retrieval extension"
                },
                {
                    "updated": "2013-07-07T12:00:0-00:00",
                    "name": "OpenStack EC2 API",
                    "links": [
                        {
                            "href": "https://github.com/openstack/identity-api",
                            "type": "text/html",
                            "rel": "describedby"
                        }
                    ],
                    "namespace": "http://docs.openstack.org/identity/api/ext/OS-EC2/v1.0",
                    "alias": "OS-EC2",
                    "description": "OpenStack EC2 Credentials backend."
                }
            ]
        }
    }


Extensions might also be queried individually by their unique alias.
This provides the simplest method of checking if an extension is
available as an unavailable extension issues an itemNotFound (404)
response.

=======  =======================  ====================================
Verb     URI                      Description
**GET**  /extensions/*``alias``*  Return details of a single extension
=======  =======================  ====================================

Normal response code(s):200, 203

Error response code(s): itemNotFound (404), badRequest (400),
identityFault (500), serviceUnavailable(503)

This operation does not require a request body.

**Example: Show extension details: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <extension xmlns="http://docs.openstack.org/common/api/v1.0"
        xmlns:atom="http://www.w3.org/2005/Atom"
        name="User Metadata Extension"
        namespace="http://docs.rackspacecloud.com/identity/api/ext/meta/v2.0"
        alias="RS-META" updated="2011-01-12T11:22:33-06:00">
        <description>Allows associating arbitrary metadata with a
            user.</description>
        <atom:link rel="describedby" type="application/pdf"
            href="http://docs.rackspacecloud.com/identity/api/ext/identity-meta-20111201.pdf"/>
        <atom:link rel="describedby" type="application/vnd.sun.wadl+xml"
            href="http://docs.rackspacecloud.com/identity/api/ext/identity-meta.wadl"
        />
    </extension>


**Example: Show extension details: JSON response**

.. code:: javascript

    {
        "extension": {
            "updated": "2013-07-07T12:00:0-00:00",
            "name": "OpenStack S3 API",
            "links": [
                {
                    "href": "https://github.com/openstack/identity-api",
                    "type": "text/html",
                    "rel": "describedby"
                }
            ],
            "namespace": "http://docs.openstack.org/identity/api/ext/s3tokens/v1.0",
            "alias": "s3tokens",
            "description": "OpenStack S3 API."
        }
    }

Extensions can define new data types, parameters, actions, headers,
states, and resources. In XML, additional elements and attributes might
be defined. These elements must be defined in the extension's namespace.
In JSON, the alias must be used. Extended headers are always
prefixed with ``X-`` followed by the alias and a dash:
(``X-RS-META-HEADER1``). Parameters must be prefixed with the extension
alias followed by a colon.

.. note::

    Applications should ignore response data that contains extension
    elements. Also, applications should also verify that an extension is
    available before submitting an extended request.

**Example: Show user details: XML response**

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <user xmlns="http://docs.openstack.org/identity/api/v2.0" enabled="true"
        email="john.smith@example.org" id="u1000" username="jqsmith">
        <metadata xmlns="http://docs.rackspacecloud.com/identity/api/ext/meta/v2.0">
            <meta key="MetaKey1">MetaValue1</meta>
            <meta key="MetaKey2">MetaValue2</meta>
        </metadata>
    </user>

**Example: Show user details: JSON response**

.. code:: javascript

    {
        "user": {
            "id": "1000",
            "username": "jqsmith",
            "email": "john.smith@example.org",
            "enabled": true,
            "RS-META:metadata": {
                "values": {
                    "MetaKey1": "MetaValue1",
                    "MetaKey2": "MetaValue2"
                }
            }
        }
    }

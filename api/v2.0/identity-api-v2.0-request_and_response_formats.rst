========================================================
OpenStack Identity API v2.0 Request and response formats
========================================================

The OpenStack Identity API supports both JSON and XML data serialization
request and response formats.

Use the ``Content-Type`` request header to specify the request format.
This header is required for operations that have a request body.

The syntax for the ``Content-Type`` header is:

.. code::

    Content-Type: application/FORMAT

Where *``FORMAT``* is either ``json`` or ``xml``.

Use one of the following methods to specify the response format:

``Accept`` header
    The syntax for the ``Accept`` header is:

    .. code::

        Accept: application/FORMAT

    Where *``FORMAT``* is either ``json`` or ``xml``. The default format
    is ``json``.

Query extension
    Add an ``.xml`` or ``.json`` extension to the request URI. For
    example, the ``.xml`` extension in the following list servers URI
    request specifies that the response body is to be returned in XML
    format:

    .. code::

        GET publicURL/servers.xml

If you do not specify a response format, JSON is the default.

If the ``Accept`` header and the query extension specify conflicting
formats, the format specified in the query extension takes precedence.
For example, if the query extension is ``.xml`` and the ``Accept``
header specifies ``application/json``, the response is returned in XML
format.

You can serialize a response in a different format from the request
format. Here are some examples.

**Example: Request with headers: JSON**

.. code::

    POST /v2.0/tokens HTTP/1.1
    Host: identity.api.openstack.org
    Content-Type: application/json
    Accept: application/xml

.. code:: javascript

    {
        "auth": {
            "tenantName": "demo",
            "passwordCredentials": {
                "username": "demo",
                "password": "devstack"
            }
        }
    }


**Example: Response with headers: XML**

.. code::

    HTTP/1.1 200 OKAY
    Date: Mon, 12 Nov 2010 15:55:01 GMT
    Content-Length:
    Content-Type: application/xml; charset=UTF-8

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <access xmlns="http://docs.openstack.org/identity/api/v2.0">
            <token issued_at="2014-01-30T15:49:11.054709"
                    expires="2014-01-31T15:49:11Z"
                    id="aaaaa-bbbbb-ccccc-dddd">
                    <tenant enabled="true" name="demo"
                            id="fc394f2ab2df4114bde39905f800dc57"/>
            </token>
            <serviceCatalog>
                    <service type="compute" name="nova">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:8774/v2/fc394f2ab2df4114bde39905f800dc57"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8774/v2/fc394f2ab2df4114bde39905f800dc57"
                                    internalURL="http://23.253.72.207:8774/v2/fc394f2ab2df4114bde39905f800dc57"
                                    id="2dad48f09e2a447a9bf852bcd93548ef"
                            />
                    </service>
                    <service type="network" name="neutron">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:9696/"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:9696/"
                                    internalURL="http://23.253.72.207:9696/"
                                    id="97c526db8d7a4c88bbb8d68db1bdcdb8"
                            />
                    </service>
                    <service type="volumev2" name="cinder">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:8776/v2/fc394f2ab2df4114bde39905f800dc57"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8776/v2/fc394f2ab2df4114bde39905f800dc57"
                                    internalURL="http://23.253.72.207:8776/v2/fc394f2ab2df4114bde39905f800dc57"
                                    id="93f86dfcbba143a39a33d0c2cd424870"
                            />
                    </service>
                    <service type="computev3" name="nova">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:8774/v3"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8774/v3"
                                    internalURL="http://23.253.72.207:8774/v3"
                                    id="3eb274b12b1d47b2abc536038d87339e"
                            />
                    </service>
                    <service type="s3" name="s3">
                            <endpoints_links/>
                            <endpoint adminURL="http://23.253.72.207:3333"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:3333"
                                    internalURL="http://23.253.72.207:3333"
                                    id="957f1e54afc64d33a62099faa5e980a2"
                            />
                    </service>
                    <service type="image" name="glance">
                            <endpoints_links/>
                            <endpoint adminURL="http://23.253.72.207:9292"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:9292"
                                    internalURL="http://23.253.72.207:9292"
                                    id="27d5749f36864c7d96bebf84a5ec9767"
                            />
                    </service>
                    <service type="volume" name="cinder">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:8776/v1/fc394f2ab2df4114bde39905f800dc57"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8776/v1/fc394f2ab2df4114bde39905f800dc57"
                                    internalURL="http://23.253.72.207:8776/v1/fc394f2ab2df4114bde39905f800dc57"
                                    id="37c83a2157f944f1972e74658aa0b139"
                            />
                    </service>
                    <service type="ec2" name="ec2">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:8773/services/Admin"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8773/services/Cloud"
                                    internalURL="http://23.253.72.207:8773/services/Cloud"
                                    id="289b59289d6048e2912b327e5d3240ca"
                            />
                    </service>
                    <service type="object-store" name="swift">
                            <endpoints_links/>
                            <endpoint adminURL="http://23.253.72.207:8080"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:8080/v1/AUTH_fc394f2ab2df4114bde39905f800dc57"
                                    internalURL="http://23.253.72.207:8080/v1/AUTH_fc394f2ab2df4114bde39905f800dc57"
                                    id="16b76b5e5b7d48039a6e4cc3129545f3"
                            />
                    </service>
                    <service type="identity" name="keystone">
                            <endpoints_links/>
                            <endpoint
                                    adminURL="http://23.253.72.207:35357/v2.0"
                                    region="RegionOne"
                                    publicURL="http://23.253.72.207:5000/v2.0"
                                    internalURL="http://23.253.72.207:5000/v2.0"
                                    id="26af053673df4ef3a2340c4239e21ea2"
                            />
                    </service>
            </serviceCatalog>
            <user username="demo" id="9a6590b2ab024747bc2167c4e064d00d"
                    name="demo">
                    <roles_links/>
                    <role name="Member"/>
                    <role name="anotherrole"/>
            </user>
            <metadata is_admin="0">
                    <roles>
                            <role>7598ac3c634d4c3da4b9126a5f67ca2b</role>
                            <role>f95c0ab82d6045d9805033ee1fbc80d4</role>
                    </roles>
            </metadata>
    </access>

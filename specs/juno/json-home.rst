..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Use JSON Home for Version/Extension discovery
=============================================

`bp json-home <https://blueprints.launchpad.net/keystone/+spec/json-home>`_

The Keystone server provides version discovery as a response to the ``GET /``,
``GET /v2.0``, and ``GET /v3`` operations. The format of this response is a
JSON document which is Keystone-specific (it's defined in the Identity API
spec). Keystone should support JSON Home, since this is a more standard way to
do REST API discovery. This will help to standardize version discovery across
the OpenStack ecosystem.

This was `discussed on the openstack-dev mailing list
<http://lists.openstack.org/pipermail/openstack-dev/2013-November/020387.html>`_.


Problem Description
===================

An application writer has a need to interact with different OpenStack services,
so she starts by writing some code to talk to the servers to discover the
version of the APIs supported by each server. She finds that all the servers
provide a different response and so has to write custom code for each
service. This is harder than it should be, since all the services should use
the same format for their version document. The format chosen for the version
document should provide a link-driven approach, such as `JSON Home`_.

With this change, Keystone will provide a JSON Home document so that it's
consistent with other services that use JSON Home.


Proposed Change
===============

The V3 Identity API spec will be changed to document that the server can
respond to a ``GET /v3`` request with a JSON Home document if the ``Accept``
header is ``application/json-home``, and also provide an example of what the
response will look like. The WADLs for the `Identity API v2`_ will be updated
similarly.

.. _`Identity API v2`: http://git.openstack.org/cgit/openstack/identity-api/tree/v2.0/src/xsd/version.xsd?id=8e9aef87e49d7b8a0a53730ad98da923588a717e

The Keystone server will be changed to check the ``Accept`` header on ``GET
/``, ``GET /v2.0``, and ``GET /v3`` request, and if the header indicates that
``application/json-home`` is the preferred format over ``application/json``,
then the server will generate a JSON Home document rather than the normal JSON
response.

Here's an example of a minimal document that only has ``/v3/users`` and
``/v3/users/{user_id}``:

.. code-block:: javascript

  {
    "resources": {
      "http://docs.openstack.org/api/openstack-identity/3/rel/users": {
        "href": "/v3/users"
      },
      "http://docs.openstack.org/api/openstack-identity/3/rel/user": {
        "href-template": "/v3/users/{user_id}",
        "href-vars": {
          "user_id": "http://docs.openstack.org/api/openstack-identity/3/param/user_id"
        },
      },
      // ...
    }
  }

The keys of the ``resources`` property are link relation types. A relationship
type needs to be chosen for the key. There are several relationship types
`registered with IANA
<http://www.iana.org/assignments/link-relations/link-relations.xhtml>`_, but
there's none designated for Identity API resources. If a group doesn't want to
register a relation with IANA (see section 4.2 "Extension Relation Types" in
`RFC 5988`_), they can use some unique URL instead. An application could
potentially fetch this URL to get information about the relationship, so we
should pick one that could potentially be used to serve up some info about what
the relationship is and describe the resource. The Nova project publishes their
XSD files at ``http://docs.openstack.org/api/openstack-compute/2/xsd/`` so
Keystone should publish its files in a similar location for consistency. For v3
resources, the relation type link will be like
``http://docs.openstack.org/api/openstack-identity/3/rel/<type>``, where
``<type>`` is like ``users``, ``user``, ``projects``, etc.

A relationship URL also has to be chosen for the parameters. For v3 parameters,
these will be like
``http://docs.openstack.org/api/openstack-identity/3/param/<parameter-type>``,
where ``<parameter-type>`` is like ``user_id``, ``project_id``, etc.

The JSON Home document that the server returns will change depending on which
extensions are enabled. The enabled extensions (that are in the pipeline) will
intercept the ``GET`` request and update the response.

There will be a resource in ``resources`` for every resource template in the v3
API (for V3), V3 extensions that are enabled, the public and admin v2 APIs and
v2 extensions that are enabled.

Here's an example of a resource for an extension, OS-EP-FILTER:

.. code-block:: javascript

  {
    "resources": {
      "http://docs.openstack.org/api/openstack-identity/3/ext/OS-EP-FILTER/1.0/rel/project_endpoint": {
        "href": "/v3/OS-EP-FILTER/projects/{project_id}/endpoints/{endpoint_id}",
        "href-vars": {
          "project_id": "http://docs.openstack.org/api/openstack-identity/3/param/project_id",
          "endpoint_id": "http://docs.openstack.org/api/openstack-identity/3/param/endpoint_id"
        }
      }
    }
  }

The relationship type for extensions will be like
``http://docs.openstack.org/api/openstack-identity/<api-version>/ext/<extension-name>/<extension-version>/rel/<resource>``.

The relationship type for a parameter used by an extension will be like
``http://docs.openstack.org/api/openstack-identity/<api-version>/ext/<extension-name>/<extension-version>/param/<param-id>``.

The extensions provided by Keystone have routers that implement the
``keystone.common.wsgi.ExtensionRouter`` class. A new ``V3ExtensionRouter``
will be created from ``ExtensionRouter`` that has code to intercept the ``GET
/v3`` request and update the response with the info for the extension. The info
for the extension will be a property that's overridden by the extension
implementation.

Alternatives
------------

None.


Data Model Impact
-----------------

None. This will not require database changes.


REST API Impact
---------------

``GET /``
~~~~~~~~~

If the ``Accept`` header is ``application/json-home``, the server will respond
with a ``200 OK`` and the JSON Home document describing the REST API, as
described in the "Proposed Change" section above.

Note that if the client is making the request with ``Accept:
application/json-home``, an old server will return the old JSON response with
``Content-Type: application/json``, so clients will have to verify that the
``Content-Type`` in the response is actually ``application/json-home`` as
expected before using the result. A server that conforms to the HTTP 1.1
specification would respond with a ``406 Not Acceptable`` error in the case
where it doesn't support the provided ``Accept`` header, so a client should
also be able to handle that response.

A client will be able to set the ``Accept`` header to a value like
``application/json; q=0.2, application/json-home`` and the server will return
JSON if it doesn't support JSON Home. Note that the Keystone server doesn't
support this today. The WebOb library that Keystone uses has support for
`Accept header handling`_, but Keystone doesn't use it for the ``Accept``
header (it's used for ``Accept-Language`` handling).

.. _`Accept header handling`: http://webob.readthedocs.org/en/latest/reference.html#accept-headers

``GET /v2.0``
~~~~~~~~~~~~~

Similar to ``GET /``, but returns the JSON Home document for only the
V2 API and extensions.

``GET /v3``
~~~~~~~~~~~

Similar to ``GET /``, but returns the JSON Home document for only the
V3 API and extensions.


Security Impact
---------------

None. The API is public info.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

python-keystoneclient should be changed to support fetching and using the JSON
Home document for discovery.


Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

When adding a new resource, or changing a resource with new arguments, the JSON
Home document will have to be updated. Extensions will have to update the JSON
Home document.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  blk-u <Brant Knudson>

Other contributors:
  <None>

Work Items
----------

1. Update the Identity V3 spec and other specs with the new ``Accept`` header
   and sample response.

2. Enhance the Keystone server so that it can process the ``Accept`` header, in
   that ``application/json; q=0.2, application/json-home`` could result in a
   JSON Home response by passing on the requested ``Accept`` header to the
   controller.

3. Change Keystone server to respond with JSON Home for ``/``, ``/v2.0``,
   ``/v3`` when the accept header is ``application/json-home``.

4. Change the v2 and v3 extensions to update the JSON Home response.

5. Write a Tempest test to verify requests for ``/``, ``/v2.0``, and ``/v3``
   with ``Accept`` set to ``application/json-home``.

6. Update python-keystoneclient to be able to use JSON Home for ``/``,
   ``/v2.0``, and ``/v3``.


Dependencies
============

None.


Testing
=======

Tempest will be changed to validate the response for ``GET /``, ``GET /v2.0``,
and ``GET /v3`` with ``Accept: application/json-home``.


Documentation Impact
====================

The documentation will need to be changed to say that Keystone supports JSON
Home.


References
==========

[0] `JSON Home
<http://tools.ietf.org/html/draft-nottingham-json-home-03>`_

[1] Nottingham, M. "Web Linking", `RFC 5988`_, October 2010

.. _`RFC 5988`: http://tools.ietf.org/html/rfc5988

[2] `Discoverable home document for APIs
<http://lists.openstack.org/pipermail/openstack-dev/2013-November/020387.html>`_
discusson on the openstack-dev mailing list.

[3] `HTTP 1.1, section 14.1 Accept Request Header
<http://tools.ietf.org/html/rfc2616#section-14.1>`_

========================================================
OpenStack Identity API v2.0 Request and response formats
========================================================

The OpenStack Identity API only supports JSON data serialization request and
response formats.

Use the ``Content-Type`` request header to specify the request format.
This header is required for operations that have a request body.

The syntax for the ``Content-Type`` header is:

.. code::

    Content-Type: application/json

Use one of the following methods to specify the response format:

``Accept`` header
    The syntax for the ``Accept`` header is:

    .. code::

        Accept: application/json

Query extension
    Add a ``.json`` extension to the request URI. For example, the ``.json``
    extension in the following list servers URI request specifies that the
    response body is to be returned in JSON format:

    .. code::

        GET publicURL/servers.json

If you do not specify a response format, JSON is the default.

If the ``Accept`` header and the query extension specify conflicting
formats, the format specified in the query extension takes precedence.
For example, if the query extension is ``.json`` and the ``Accept``
header specifies ``application/xml``, the response is returned in JSON
format.

You can serialize a response in a different format from the request
format. Here are some examples.

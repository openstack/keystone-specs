===============================================
OpenStack Identity API v2.0 Content compression
===============================================

Request and response body data may be encoded with gzip compression in
order to accelerate interactive performance of API calls and responses.
This is controlled using the ``Accept-Encoding`` header on the request
from the client and indicated by the ``Content-Encoding`` header in the
server response. Unless the header is explicitly set, encoding defaults
to disabled.

**Compression headers**

=================  ================  =====
Header type        Name              Value
=================  ================  =====
HTTP/1.1 Request   Accept-Encoding   gzip
HTTP/1.1 Response  Content-Encoding  gzip
=================  ================  =====


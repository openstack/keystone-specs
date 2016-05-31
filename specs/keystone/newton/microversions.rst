..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============
Microversions
=============

`bp microversions <https://blueprints.launchpad.net/keystone/+spec/microversions>`_


Clone the Nova approach to using Microversions to provide API access
to features.


Problem Description
===================

Nova uses a framework called ‘API Microversions’ for allowing changes
to the API while preserving backward compatibility. The basic idea is
that a user has to explicitly ask for their request to be treated with
a particular version of the API. So breaking changes can be added to
the API without breaking users who don’t specifically ask for it. This
is done with an HTTP header X-OpenStack-Nova-API-Version which is a
monotonically increasing semantic version number.

If a user makes a request without specifying a version, they will get
the DEFAULT_API_VERSION as defined in nova/api/openstack/wsgi.py. This
value is currently 2.1 and is expected to remain so for quite a long
time.

There is a special value latest which can be specified, which will
allow a client to always receive the most recent version of API
responses from the server.

Keystone has similar requirements to Nova in wanting to be able to introduce
changes that change the API behaviour of individual APIs, but currently does
not formally support the concept of microversions.


Proposed Change
===============

Implement the same microversion approach in keystone. In fact, keystone has
had a form of microversions since the introduction of 3.0. At each release, the
microversion of the API is defined in the Identity Specification (Mitaka was
version 3.6). This microversion is also returned as part of the versions
response from the / and /v3 API calls. What keystone does not currently allow
is for a client to request a particular microversion - you always get the
latest one. Hence the change proposed here is to support this client-server
interrogation, as well as the server to support more than version at a time.

Microversions are implemented in the API through the supporting the standard
OpenStack version HTTP header 'X-OpenStack-API-Version', with a service
type of 'identity'.  For example::

'X-OpenStack-API-Version': identity 3.7

This approach is laid out in the cross-project specification:
<https://specs.openstack.org/openstack/api-wg/guidelines/microversion_specification.html>`_.

This header is accepted by keystone so a client can indicate which version of
the API it wants to use for communication, and likewise for keystone to
indicate which version it is using for communication.

For the Identity API, if no HTTP header is supplied, microversion 3.6 of the v3
API is used (stable/mitaka) - i.e. the last version before microversions were
introduced. If an invalid version is specified in the HTTP header, an HTTP 406
Not Acceptable is returned. If the special 'latest' version is specified,
keystone will use its most recent version (3.7 for Newton). Asking for a
specific microversion before 3.6 will not be supported (and any such request
will be treated as an invalid version).

Use Cases
---------

The following represents the keystone equivalent set of the user cases that are
supported in the nova approach, in this case between keystone and
python-keystoneclient.

For the purposes of definition, we will use the term "KeystoneV3" to refer to
a version of keystone that predates microversions and has no knowledge of them.
Likewise, we will use the term "KeystoneV3Microversioned" to refer to a version
of keystone that includes support for microversions. Microversioning is not
supported on the keystone v2.0 API, only the v3 API can have its version
negotiated.

In the use case below, for python-keystoneclient, the label "old client" refers
to a version of python-keystoneclient which doesn't support microversions and
"new client" to version of python-keystoneclient which does supports it.

Use Case 1: Old Client communicating with KeystoneV3
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is exactly the same behaviour that was seen prior to the introduction
of microversions - no change to either the client or server is required
for this case.

Use Case 2: Old Client communicating with KeystoneV3Microversioned
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is where keystone is updated to a new version that supports microversions,
but an old client is used to communicate to it.

* The client makes a connection to keystone, not specifying the HTTP header
  X-OpenStack-API-Version
* Keystone does not see the X-OpenStack-API-Version HTTP header
* Keystone communicates using the 3.6 microversion of the v3 API(stable/mitaka)
  and all communication with the client uses that version of the interface.

Use Case 3A: New Client communicating with KeystoneV3 (not user-specified)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the where the user does not request a particular microversion to a
new client that support microversions and tries to communicate with an old
keystone.

* The user does not specify the microversion to use in communication with
  the client.
* The client makes a connection to keystone and asks for supported API
  versions (using the GET /v3/ API)
* Keystone doesn't look for, or parse the HTTP header. It just returns the
  versions json response.
* The client checks versions info, and selects the latest non-microversioned
  API level supported by the server (which by definition can be no later
  than 3.6). Note that this is different to the nova approach (where if no
  version is specified, they fail) - this is justified by the fact that our
  client already will select the latest version in this scenario.

Use Case 3B: New Client communicating with KeystoneV3 (user-specified)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the where the user requests a particular microversion to a
new client that support microversions and tries to communicate with
KeystoneV3.

* The user specifies a microversion that is valid for the client.
* The client makes a connection to KeystoneV3, supplying a
  X-OpenStack-API-Version HTTP header
* Keystone doesn't know to look for, or parse the HTTP header. It communicates
  using whatever the latest version it supports which, by definition, will be
  no later than 3.6.
* The client does not receive a X-OpenStack-API-Version header in
  the response, and from that is able to assume that the version of keystone
  that it is talking to does not support microversions. That is, it is using
  a version of the REST API that predates KeystoneV3Microversioned.
* The client informs the user that it cannot communicate to keystone using that
  microversion and exits. (Conceptually we could allow the specific case of
  asking for 3.6 against a Mitaka keystone to work here, but it does not seem
  worth supporting that corner case).

Use Case 3C: New Client communicating with KeystoneV3 (backward-compatibility)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the way to use KeystoneV3 via new client.

* The user specifies an identity api version of "3" (for example, as is done
  by the openstackclient today)
* The client checks that minor part of version is zero and hence assumes that a
  microversion is not used.
* The client makes a connection to KeystoneV3, without adding a
  X-OpenStack-API-Version HTTP header.
* Keystone doesn't know to look for, or parse the HTTP header. It communicates
  using whatever the latest version it supports which, by definition, will be
  no later than 3.6.
* The client doesn't look for, or parse the HTTP header, it knows that
  microversions are not being used.
* The client processes received data, display it to user and exits.

Use Case 4: New Client, user specifying an invalid version number
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the case where a user provides as input to a new client an invalid
microversion identifier, such as 'spam', 'l33t', or '1.2.3.4.5'.

* The user specifies a microversion to the client that is invalid.
* The client returns an error to the user, i.e. the client should provide
   some validation that a valid microversion identifier is provided.

A valid microversion identifier must comply with the following regex:

  ^([1-9]\d*)\.([1-9]\d*|0|latest)$

Examples of valid microversion identifier: '3.7', '3.21', '3.latest',...

Use Case 5: New Client/KeystoneV3Microversioned: Unsupported keystone version
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the case where a new client requests what was once a valid
microversion, but is now older than the KeystoneV3Microversioned can handle.
Although today this won't be possible (since both support the same set of
versions), in the future this may be required).

* The client makes a connection to keystone, supplying 3.x as the requested
  microversion.
* Keystone responds with a 406 Not Acceptable.
* As the client does not support a version supported by keystone, it cannot
  continue and reports such to the user.

Use Case 6: New Client/KeystoneV3Microversioned: Unsupported Client version
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the case where a new client requests a version that is newer than
the KeystoneV3Microversioned can handle.  For example, the client support
microversions 3.8 to 3.9, and the particular keystone supports versions 3.7.

Steps are the same as Use Case 5.

Use Case 7: New Client/KeystoneV3Microversioned: Compatible Version
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the case where a new client requests a version that is supported
by KeystoneV3Microversioned. For example, the client supports microversions 3.6
to 3.7, as does the server.

* The client makes a connection to keystone, supplying 3.7 as the requested
  microversion.
* As keystone can support this microversion, it responds by sending back a
  response of 3.7 in the X-OpenStack-API-Version HTTP header.

Use Case 8: New Client/KeystoneV3Microversioned: Version request of 'latest'
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the case where a new client requests a version of 'latest' from a
KeystoneV3Microversioned.

* The user specifies 'latest' microversion is to be used.
* The client makes a connection to keystone and asks for supported API
  versions (using the GET /v3/ API)
* Keystone doesn't look for, or parse the HTTP header. It just returns the
  versions json response.
* The client checks API version info and makes conclusion that current version
  supports microversions.
* The client chooses the latest version supported both by client and server
  sides(via "version" and "min_version" values from API version response) and
  makes a connection to keystone, supplying selected version in the
  X-OpenStack-API-Version HTTP header

Use Case 9: New Client/KeystoneV3Microversioned: Version not specified
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is the where the user does not request a particular microversion to a
new client that support microversions and tries to communicate with a
KeystoneV3Microversioned.

* The user does not specify the microversion to use in communication with
  the client.
* The client makes a connection to keystone and asks for supported API
  versions (using the GET /v3/ API)
* Keystone doesn't look for, or parse the HTTP header. It just returns the
  versions json response.
* The client checks versions info, and selects the latest non-microversioned
  API level supported by the server (which by definition will be 3.6). Note
  that this is different to our current client where, prior to these changes,
  the absolute latest version would have been selected.

Specific changes
================
The python identity API in keystoneclient should be extended to
include major and minor parts of version. It should look like:

* "X.Y" - "X" and "Y" accept numeric values. The client will use it to
  communicate with keystone.
* "X.latest" - "X" accepts numeric values. The client will use the "latest"
  (see `latest-microversion`_ for more details) supported both by client and
  server sides microversion of "X" Major version.
* "latest" - The client will use the latest major version known by client and
  "latest" (`latest-microversion`_) microversion supported both by client and
  server sides.

   "X" is a major part and "Y" is a minor one

The requested microversion (when it specified) should be used (unless
the client cannot support that version). The client will always
request a specific microversion in its communication with the
server. 'X.latest' is purely a signal from a python consumer that it
wants negotiation of the maximum mutually-supported version between
the server and client.

python-keystoneclient as a CLI tool
-----------------------------------
Since this is deprecated, no changes will be made to this.

python-openstackclient CLI tool
-------------------------------
Microversions should be specified with a major API version, using the
existing --os-identity-api-version option. Today this only contains the major
version (i.e. --os-identity-api-version=3), but can now be specified, for
example, as --os-compute-api-version=3.7. This value will then be passed to
python-keystoneclient.

A user may also specify --os-compute-api-version="None" which indicates that
client should use should use default major API version without microversion
(this is provided to be compatible with nova's approach and is equivalent to
setting this to 3).

Help messages should display all variations of commands, sub-commands and their
options with information about supported versions(min and max).

Once v3 is supported as the default for the identity API, the actual default
setting should be "3.latest" - so that a given version of the client will use
the latest version supported by the client library and server.

python-keystoneclient as a Python lib (keystoneclient.client entry point)
-------------------------------------------------------------------------
Module ``keystoneclient.client`` is used as entry point to
python-keystoneclient inside other python libraries.

``keystoneclient.client.Client`` already accepts a version tuple (X, Y) and
this will be used to communicate any requested microversion, in which case the
client should add the header X-OpenStack-API-Version to each server
call and validate response includes equal header too, which means the API side
supports the required microversion.

.. _latest-microversion:

"latest" microversion
~~~~~~~~~~~~~~~~~~~~~
"latest" microversion is a maximum version. Despite the fact that Identity API
accepts value "latest" in the header, the client doesn't use this ability.
The client discovers the "latest" microversion supported by both API, client
sides and uses it in communication with Nova-API.

Discover should be processed by following steps:

* The client makes one extra call to Identity API - list all versions;
* The client checks that current version supports microversions by checking
  values "min_version" and "version" of current version. If current version
  doesn't support microversions("min_version" and "version" are empty),
  the client raises an exception with this information.
* The client chooses latest microversion supported by both keystoneclient and
  Identity API.

NOTE: To decrease number of extra calls, the client should cache discovered
versions. Since different methods/API calls can have different "latest"
versions, each discovered versions should be cached with related method.

python-keystoneclient from developer point of view : adding new microversions
-----------------------------------------------------------------------------
Each "versioned" method of ResourceManager should be labeled with a specific
decorator. Such decorator should accept two arguments: start version and end
version (optional). Example:

.. code-block:: python

  from keystoneclient import api_versions
  from keystoneclient import base

  class SomeResourceManager(base.Manager)
      @api_version(start_version='3.7', end_version='3.7')
      def show(self, req, id):
          do work and return results specific to version 3.7

      @api_versions.wrap(start_version='3.8')
      def show(self, req, id):
          do work and return results specific to version 3.8 onwards

A similar approach will be taken in openstackclient, to version the actual
commands (and their arguments).

Alternatives
------------

There are many possible approaches to this.  The one proposed consistent with
other projects, which provides distinct advantages for operators.

Alternatives are:

* We only make purely additive changes to the API, which cannot bleed over into
  other existing APIs (this could be very restrictive), and/or
* Only make breaking changes at major API levels (e.g. v3, v4, v5 etc.), where
  we accept a multi-year change over period.


Security Impact
---------------

None


Notifications Impact
--------------------

None


Other End User Impact
---------------------

Clients that wish to use new features available over the REST API added since
the 'Mitaka' release will need to start using this HTTP header.  The fact that
new features will only be added in new versions will encourage them to do so.


Performance Impact
------------------

None


Other Deployer Impact
---------------------

None


Developer Impact
----------------

Any future changes to the Identity REST API (whether that be in the request or
any response) *must* result in a microversion update, and guarded in the code
appropriately.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Henry Nash (henry-nash)
Other contributors:
  Adam Young (ayoung)
  Morgan Fainberg (notmorgan)

Work Items
----------

Dependencies
============

None.  Many will depend on this.

Documentation Impact
====================

This change will resonate through the docs.

References
==========

[1] https://specs.openstack.org/openstack/api-wg/guidelines/microversion_specification.html

[2] http://git.openstack.org/cgit/openstack/nova-specs/tree/specs/kilo/implemented/api-microversions.rst

[3] http://docs.openstack.org/developer/nova/api_microversions.html

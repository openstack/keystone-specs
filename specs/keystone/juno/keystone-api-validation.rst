..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
API Validation
==============

`bp api-validation <https://blueprints.launchpad.net/keystone/+spec/api-validation>`_

Currently, Keystone has different implementations for validating
request bodies. The purpose of this blueprint is to track the progress of
validating the request bodies sent to the Keystone server, accepting requests
that fit the resource schema and rejecting requests that do not fit the
schema. Depending on the content of the request body, the request should
be accepted or rejected consistently regardless of the resource the request
is for.


Problem Description
===================

Currently Keystone doesn't have a consistent request validation layer. Some
resources validate input at the resource controller and some fail out in the
backend. Ideally, Keystone would have some validation in place to catch
disallowed parameters and return a validation error to the user.

The end user will benefit from having consistent and helpful feedback,
regardless of which resource they are interacting with.

Use Case: As an End User, I want to observe consistent API validation
regardless of the backend being used and values passed to the Keystone server.


Proposed Change
===============

One possible way to validate the Keystone API is to use jsonschema
(https://pypi.python.org/pypi/jsonschema). A jsonschema validator object can
be used to check each resource against an appropriate schema for that
resource. If the validation passes, the request can follow the existing flow
of control to the resource manager to the backend. If the request body
parameters fail the validation specified by the resource schema, a validation
error will be returned from the server.

Example:
"Invalid input for field 'email'. The value is 'some invalid email value'.

We can build in some sort of truncation check if the value of the attribute is
too long. For example, if someone tries to pass in a 200 character email
address we should check for that case and then only return a useful message,
instead of spamming the logs. Truncating some really long email address might
not help readability for the user, so return a message to the user with what
failed validation.

Example:
"Invalid input for field 'email'."

Some notes on doing this implementation:

* Common parameter types can be leveraged across all Keystone resources. An
  example of this would be as follows::

    from keystone.common.validation import parameter_types
    <snip>
    CREATE = {
        'type': 'object',
        'properties': {
            'name': parameter_types.name,
            'description': parameter_types.description,
            'enabled': parameter_types.boolean,
            'url': parameter_types.url
        },
        'required': ['name'],
        'additionalProperties': True,
    }

* The validation can take place at the controller layer.

* Initial work will include capturing the Identity API Spec for existing
  resources in a schema. This should be a one time operation for each
  major version of the API. This will be applied to the Identity V3 API.

* The current implementation up for review uses a method decorator in the
  resource controller [1]. This is a fairly simple change and doesn't clutter
  the existing controller code.

* When adding a new extension to Keystone, the new extension must be proposed
  with its appropriate schema.

There are a couple notes on how to deal with different backend constraints.
The validator will have to honor the Identity API Spec but it will also have
to account for the cases specific to the backend. Example, the SQL backend
allows for 255 character user names but the LDAP backend doesn't have a
restriction against the length of user names. In this case, we could do the
following:

* Validate against the least common denominator at the controller layer,
  unlimited in this example.

* Have the backend call back to the validator, giving it a value to validate
  with. Example, the SQL backend understands the schema, so when asked the
  limit of user names, it could return 255.


Alternatives
------------

Another alternative would be to map the properties from the incoming request
to Python objects and enforce the contract that way. This might be a tougher
choice since Python is not stictly typed.

For the time being, jsonschema will fill this requirement. If at some point
jsonschema no longer meets the needs of validating requests we can look into
another framework, or consider building our own validation framework, specific
to the use cases we need.

`Voluptuous <https://github.com/alecthomas/voluptuous>`_ might be another
option for input validation.

There have been discussions about using
`Pecan <http://pecan.readthedocs.org/en/latest/>`_ as the web frame work for
Keystone. Pecan does offer input validation in conjunction with WSME. If/When
Keystone moves over to using Pecan, the validation layer may be refactored
then. This implementation should be sufficient and work with Pecan until it is
decided if validation will live with Pecan or not.


Data Model Impact
-----------------

The incoming request could be represented as an object, in
which case the request object would have the jsonschema validator as an
attribute.

This blueprint shouldn't require a database migration or schema change.

There is discussion about implementing the request as an object, which
jamielennox has posted in a
`review <https://review.openstack.org/#/c/92031/>`_.


REST API Impact
---------------

This blueprint shouldn't affect the existing API. In the event that it does,
it will be correcting the API to follow the Identity API Spec, if possible.
See the `API Change Guidelines <https://wiki.openstack.org/wiki/APIChangeGuidelines#Generally_Not_Acceptable>`_.
In the event a bug is discovered in a stable release that has already shipped,
we will need to address in a case-by-case basis and update the API spec
accordingly.


Security Impact
---------------

The output from the request validation layer should not compromise data or
expose private data to an external user. Request validation should not
return information upon successful validation. In the event a request
body is not valid, the validation layer should return the invalid values
and/or the values required by the request, of which the end user should know.
The parameters of the resources being validated are public information,
described in the Identity API spec, with the exception of private data. In the
event the user's private data fails validation, a check can be built into the
error handling of the validator to not return the actual value of the
private data.

jsonschema documentation notes security considerations for both schemas and
instances:
http://json-schema.org/latest/json-schema-core.html#anchor21


Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Changes required for request validation do not require any locking mechanisms.


Other Deployer Impact
---------------------

None


Developer Impact
----------------

This will require developers contributing new extensions to Keystone to have
a proper schema representing the extension's API.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
ldbragst (Lance Bragstad <ldbragst@us.ibm.com> <lbragstad@gmail.com>)

Other contributors:
jamielennox (Jamie Lennox <jamielennox@redhat.com>)

Work Items
----------

1. Initial validator implementation, which will contain common validator code
   designed to be shared across all resource controllers validating request
   bodies.
2. Introduce validation schemas for existing core API resources.
3. Introduce validation schemas for existing API extensions.
4. Enforce validation on proposed core API additions and extensions.


Dependencies
============

None

Testing
=======

Tempest tests can be added as each resource is validated against its schema.
These tests should walk through invalid request types.

We can follow some of the validation work already done in the Nova V3 API:

* `Validation Testing <http://git.openstack.org/cgit/openstack/tempest/tree/etc/schemas/compute/flavors/flavors_list.json?id=24eb89cd3efd9e9873c78aacde804870962ddcbb>`_

* `Negative Validation Testing <http://git.openstack.org/cgit/openstack/tempest/tree/tempest/api/compute/flavors/test_flavors_negative.py?id=b2978da5ab52e461b06a650e038df52e6ceb5cd6>`_

Negative validation tests should use tempest.test.NegativeAutoTest

Documentation Impact
====================

None

References
==========

[1] [Existing Work] (https://review.openstack.org/#/q/status:open+project:openstack/keystone+branch:master+topic:validator,n,z)

Useful Links:

* [Understanding JSON Schema] (http://spacetelescope.github.io/understanding-json-schema/reference/object.html)

* [Nova Validation Examples] (http://git.openstack.org/cgit/openstack/nova/tree/nova/api/validation)

* [JSON Schema on PyPI] (https://pypi.python.org/pypi/jsonschema)

* [JSON Schema core definitions and terminology] (http://tools.ietf.org/html/draft-zyp-json-schema-04)

* [JSON Schema Documentation] (http://json-schema.org/documentation.html)

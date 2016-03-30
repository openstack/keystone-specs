..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Add names to list assignments
=============================

`bp list-assignment-with-names <https://blueprints.launchpad.net/keystone/+spec/list-assignment-with-names>`_

Optionally allow a caller of the list assignment API to ask for the
entities returned to include their names.

Problem Description
===================

The current list assignment API returns just the IDs of the entities. In
general, for this to be useful, a client would typically have to convert these
IDs into the names of the entities. It would be much more efficient if this
could be done in the server as part of the API.

Proposed Change
===============

Support an additional query parameter `include_names` to the list
assignment API. If specified as `true`, then each of the entities returned
will include the name. For entities who's name is only unique within a domain,
the domain name is also returned. The ability to list assignments by entity
name is also supported.

While we could return all the attributes of each entity, given the potential
large number of elements in a collection, we only include the name. The `id` is
also still returned so that if the caller needs the full entity they can obtain
it.

Alternatives
------------

Leave things the way they are.

Data Model Impact
-----------------

None

REST API Impact
---------------

None, other than to support the additional query parameter.

Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

There is obviously a potential performance impact for large collections.
This will be minimized where possible with efficient SQL coding.

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
    henry-nash


Work Items
----------

- Add manager/driver support for names
- Add controller for names
- Add keystoneclient library support for names
- Add openstack cli support for names

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Changes to user documentation to describe new API.

References
==========

None

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Split Identity Middleware into a Distinct Package
=================================================

`bp split-middleware
<https://blueprints.launchpad.net/keystone/+spec/split-middleware>`_

Keystone has a sufficient quantity of middleware to justify a separate repo.
These will come from multiple projects, to include the Keystone server and
python-keystoneclient repositories. This split is specifically to address
a few major requests:

* Limit the dependencies of python-keystoneclient. Some of the middleware
  dependencies do not make sense to require of a CLI utility or an API
  library (e.g. memcache).

* Allow for separate release scheduling (and isolation of changes) between
  server requirements (e.g. the middleware) and the CLI/client library. The
  separate packaging and release schedule allows for a narrower scope of
  concern when updating the middleware (no accidental impact to the public
  interfaces of the client library/CLI).

* Consolidate all OpenStack Identity-specific middleware to a single
  repository.

Problem Description
===================

Most of the middleware currently lives in python-keystoneclient. However, the
middleware code pulls in dependencies specific to servers, and that are not
appropriate for command line clients and other integrated scripts. There are
sufficiently different use cases that they should be their own repositories and
python packages.

Proposed Change
===============

Create a new repository called keystonemiddleware and use the
oslo-incubator graduation utilities to copy the data from
python-keystoneclient/middleware and keystone/middleware while
maintaining the commit history. The tests for the middleware code will
also be migrated (maintaining history as possible). Only middleware
from Keystone that is used by external services will be migrated
(e.g. ec2token).

Once the keystonemiddleware package is released the middleware
in the python-keystoneclient will be frozen (security fixes only,
no new features) and all new development would be done against the code
in the new middleware package. Middleware originally from Keystone
will be made available (via import) at the old location and wrapped
with a deprecation message instead of frozen.

The old middleware code would remain deprecated and available for
at least two full OpenStack release cycles. Options for removal
will be considered during the L and later development cycles.

The new middleware package will be released independantly of
the OpenStack named releases (similar to the python-keystoneclient
library).

The inital release of the middleware package will be version 1.0.0
(using semver for versioning scheme).

Alternatives
------------

* Split Keystoneclient up into multiple libraries

  * common code (e.g. session, cms), this could be an oslo-lib

  * middleware (auth_token, ec2, s3)

  * client code (left in the keystonclient package)

Data Model Impact
-----------------

None

REST API Impact
---------------

None

Security Impact
---------------

* Two versions of middleware will need to receive security fixes until
  the deprecated middleware is removed.

Notifications Impact
--------------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

Deployers will need to update the paste-ini configurations to reference the
new location of the middleware. The old middleware will remain available
within the ``python-keystoneclient`` package for at least two full OpenStack
release cycles. The old middleware would receive only security fixes while
it remains in the ``python-keystoneclient`` package.

Developer Impact
----------------

Developers will need to be aware of which repo to submit code changes against.

Implementation
==============

Assignee(s)
-----------

morganfainberg

ayoung

Work Items
----------

* Create new repo.

* Copy middleware code and tests from python-keystoneclient to new repo using the
  oslo-incubator graduation utilities to maintain the change history.

* Copy middleware (used by external services (e.g. ec2token)) code and tests from
  keystone to the new repo using the oslo-incubator graduation utilities to
  maintain the change history.

* Change keystoneclient and Keystone middleware so that it prints out a deprecated
  message if it's used. This includes updating any deprecation messages to point
  to the new package.

* Release an initial version of the keystonemiddleware package (to be performed
  before deprecation updates).

* Update global requirements to add the new keystonemiddleware package.

* Update services and paste-ini example files to use the new keystonemiddleware
  package.


Dependencies
============

None

Testing
=======

* Middleware specific unit tests will be moved to the new repository.

* Gate configuration will ensure a full integration test run for changes to
  the new repository.

Documentation Impact
====================

Docs will need to be updated as far as where to look for keystone middleware

References
==========

None

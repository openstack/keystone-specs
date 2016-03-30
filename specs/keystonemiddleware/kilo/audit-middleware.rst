..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Adding audit middleware to keystonemiddleware
=============================================

`Auditing middleware
<https://blueprints.launchpad.net/keystonemiddleware/+spec/audit-middleware>`_

The pyCADF library contains middleware which enables the ability to audit API
calls to a given service. The audit middleware utilizes the identity data
provided by the ``auth_token`` middleware.

Problem Description
===================

Auditing is heavily tied to identity but currently the audit middleware exists
in pyCADF library while the identity middleware are contained in
``openstack/keystonemiddleware``. This requires deployers to explicitly pull in
multiple dependencies. Since there's a logical association between them, the
middleware should be grouped accordingly.

Proposed Change
===============

Currently, the audit middleware exists in `pyCADF library
<https://github.com/openstack/pycadf/blob/fa802a753d00b4e61eebbc7360caecffba3d7852/pycadf/middleware/audit.py>`_
the proposed solution is to move this middleware into ``keystonemiddleware``.
This solution brings in a dependency on ``oslo.messaging`` as the current audit
middleware places audit events to message queue. It also has a dependency on
pyCADF to generate audit events.

Alternatives
------------

Two alternatives:

* Keep things as-is. If the user wants to audit, they should pull in pyCADF
  and ``notifiermiddleware`` and add audit middleware.

* Pull in audit middleware from pyCADF but leave off ``oslo.messaging``
  dependency. Notifications can be delegated to ``notifiermiddleware`` but
  requires a change to ``notifiermiddleware`` to properly audit both request
  and response.

Security Impact
---------------

None

Notifications Impact
--------------------

The proposed solution will have the middleware send two notifications per API
request: one for the request and another for the response. It can be configured
to only audit certain API requests (for example, just GET requests) to minimize
notifications.

Other End User Impact
---------------------

Users need to consume ``audit`` middleware from a python package
(``keystonemiddleware.audit``).

Documentation will be moved from the `old location
<http://docs.openstack.org/developer/pycadf/middleware.html>`_ to a new
location in ``keystonemiddleware``.

Performance Impact
------------------

This will create more load on message queue if enabled. This audit filter is
optional.

Other Deployer Impact
---------------------

If enabled, deployers need to enable notifications in the service where
middleware is being configured. After that, they can add audit middleware to
WSGI pipeline as described in documentation.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  chungg

Other contributors:
  None

Work Items
----------

* Move audit middleware to ``keystonemiddleware`` and make appropriate code
  changes to get it working, providing backwards compatibility in pyCADF.

* Update ``keystonemiddleware`` docs to include middleware configuration docs.

Dependencies
============

* Need pyCADF and oslo.messaging libraries

Documentation Impact
====================

Copy documentation for enabling middleware:
http://docs.openstack.org/developer/pycadf/middleware.html

References
==========

None

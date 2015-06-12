..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Data Driven Assignment Unit Testing
===================================

`bp data-driven-tests <https://blueprints.launchpad.net/keystone/+spec/data-driven-tests>`_


Improve our ability to do detailed role assignment testing using a data
driven test formats as opposed to acres of method calls.


Problem Description
===================

In many ways, the core function of keystone is to store and provide access to
the role assignments on the artifacts that keystone understands (i.e. projects,
domains). As keystone deployments scale up, we are continually adding more
advanced support for assignment access, as well as pushing more and more
intelligence into the drivers for efficiency (e.g. filtering). The use of
inheritance and project hierarchies only makes this even more complicated. As
we expand this functionality, the core complexity in the assignments engine
actually comes down to the list role assignments method. This single backend
method will be called for a wide variety of uses (CRUD API response, token
generation, federation mapping etc.).

Given how crucial this support will be, we need to ensure our unit testing
keeps pace with this. In the past we have written our unit testing simply
as pages of method calls to the backends, but this becomes unwieldy and
hard to understand and maintain given the numerous filtering options and
scenarios that are now supported.

It would be preferable if we had a more compact and see-at-a-glance method
of writing the increased testing required in this area.

Proposed Change
===============

While we could go ahead and boil the ocean here and write a full on natural
language test engine, it is proposed that we use a simple data driven
dictionary approach for defining a test plan. For example a very simple test
plan might be::

        test_plan = {
            'entities': {'domain': [{'contents': {'user': 1, 'group': 1,
                                                  'project': 1}}],
                         'role': 2},
            'assignments': [{'user': 0, 'role': 0, 'domain': 0},
                            {'user': 0, 'role': 1, 'project': 0}],
            'tests': [
                {'params': {},
                 'results': [{'user': 0, 'role': 0, 'domain': 0},
                             {'user': 0, 'role': 1, 'project': 0}]},
                {'params': {'role': 1},
                 'results': [{'user': 0, 'role': 1, 'project': 0}]}
            ]
        }

The above says:

* Create a domain entity, with 1 user, group and project within it. Also
  create 2 roles.  All entities created can be referred to via an index,
  that states from zero and is incremented in the order they appear in the
  test plan
* Make 2 assignments, i.e. give user 0 the role 0 on domain 0 and also give
  that same user role 1 on project 0.
* Run two list role assignment tests (by calling the assignment manager call
  list_role_assignments), the first test with no parameters and checking that
  it returns the correct two assignments, and a second test passing in a filter
  for only returning assignments with role index 1.

The test helper that drives the above test plan obviously turns entity indexes
into real entity IDs for calling the actual driver method.

An implementation of the above test plan format and helper was actually
posted in the Kilo cycle as a series of patches to support the migration of
filtering on the list assignment method into the backend. In the end we
deferred this support to Liberty, so now is the time to decide on the type of
testing we want for this.

The following set of Kilo patches added increasing support for this data driven
methodology and actually found numerous defects during the development of
the proposed new list role assignment backend method. It is clear that
something like this is required.

https://review.openstack.org/#/c/149178/
https://review.openstack.org/#/c/151623/
https://review.openstack.org/#/c/151962/
https://review.openstack.org/#/c/154302/
https://review.openstack.org/#/c/153897/
https://review.openstack.org/#/c/154485/

The above patches implement adding support for:

* Using existing entities
* "Effective mode" in list role assignments
* Groups
* Inheritance and project hierarchies

Alternatives
------------

As discussed above, we could write a more complex natural language processor
test system - but since this is a test tool for internal development and we
need to balance how much code is in the tool vs the tests it is replacing.
There are a couple of issues here.  The first is that ideally we
don't want to have a test helper that is so complex that it itself needs to be
tested in its own right. Second, we should be conscious of performance. Being a
pure data defined test approach, the current proposal is really creating a set
of directed loops straight from the data supplied. This is often what testers
do manually anyway - so there should be little impact on test performance with
this approach. A solution that required more processing of a test plan in order
to turn it into manager calls would likely have more of an impact on
performance.

We could also make the test plan more generic - for instance, maybe there are
other backend methods that we might want to apply that too. It is proposed that
we solve the problem at hand, then consider any such changes in the future.

Data Model Impact
-----------------

None

REST API Impact
---------------

None

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

The performance impact of running tests should be minimal, since we are
basically using a dictionary to drive a set of directed loops.

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

The code of the above is already written, so we simply need to rebase it all.

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

None

References
==========

None

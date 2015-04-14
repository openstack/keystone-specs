..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Object Dependency Lifecycle
===========================

`bp object-dependency-lifecycle <https://blueprints.launchpad.net/keystone/+spec
/object-dependency-lifecycle>`_

The way dependency injection (DI) is currently implemented in Keystone is not
really using the "dependency injection" pattern. This spec is focused on
improving the internal-to-Keystone object lifecycle (including dependencies).

This is based off of a Kilo Summit session:
`Pre-session Reading <https://www.morganfainberg.com/blog/2014/10/21
/openstack-kilo-summit-pre-summit-thoughts/#objectlifecycle>`_

Problem Description
===================

Goals:
 * What should the dependency injection pattern in Keystone look like?
 * How do we deal with Optional dependencies?
 * What is the lifecycle of objects (controllers, managers, etc.)?

Motivation
 * Remove the magic of just adding attributes on to an object without really
   having them defined in the API
 * Optional dependencies are currently not implemented in a very
   object oriented way; we use the existence of the dependency as a sort of
   feature flag
 * Better seams for testing; specifically optional dependencies
 * DI is about dynamically building object graphs where different
   implementations of some interface can be used. We don't really do that.

Constraints:
 * Keep APIs as backward compatible as possible (drivers for sure, managers and
   controllers to a lesser extent)
 * Compatibility with `Stevedore`_. We may be looking to use Stevedore for
   dynamically loading drivers and we should make sure we don't do anything
   that would make that harder or impossible.

What is dependency injection?
-----------------------------

Wikipedia has a `good description of dependency injection
<http://en.wikipedia.org/wiki/Dependency_injection>`_. In short the pattern
is a way of separating the code that implements the business logic from the
code that creates objects. Instead of creating dependencies, objects will
expect those dependencies to be passed to its constructor. This allows the
construction of object graphs (the tree of objects) to vary without changes
to the objects themselves.


Proposed Change
===============

1. Phase I - Manual DI

 * Go through and fix classes to expect dependencies to be provided in via
   their __init__
 * This will remove the magic and make dependencies a part of the actual
   object definition
   (proof of concept http://paste.openstack.org/show/129171/)
 * Construct object graphs in a new uber factory
   (proof of concept http://paste.openstack.org/show/129179/)

  * This keeps the object lifecycle of the injected objects as singleton
  * This also keeps a global registry

 * Keep singleton scope on the current object graphs
 * Implement optional dependencies using the __init__ as optional kwargs

  * This will keep the internal code of the objects the same. We would still
    'if self.mydep'

 * Pitfalls:

  * We need to add something to the wire up next things. Right now through the
    magic of the decorator this just happens.

2. Phase II - Evaluate Manual DI

 * Is there a need for a framework?

  * Is there enough of a pain point to use a DI framework?
  * Is there enough need of some of a DI framework's features to use it?
  * Keystone may not be complicated enough to need anything more.

 * There are a few existing frameworks like inject and snake-guice
 * Do we need objects that are scoped differently? request scoped, other...

3. Phase III - Fix up the model of optional dependencies

 * I think that optional dependencies should be implemented on top of the
   existing object using composition. Having code doing 'if self.dep' is a
   bad smell.

  * e.g. The optional thingee_api dependency in
    keystone.assignment.core.Manager could be implemented as a wrapper around
    the backend using a decorator pattern.
  * Other optional dependencies may have to be handled differently

Alternatives
------------

* We could go all in using a complicated framework with too many features.
* We could do nothing and continue to grow the magic.

Security Impact
---------------

None. This spec is about internal modifications to the code structure.

Notifications Impact
--------------------

None. This spec is about internal modifications to the code structure.

Other End User Impact
---------------------

None. This spec is about internal modifications to the code structure.

Performance Impact
------------------

None. This spec is about internal modifications to the code structure.

Other Deployer Impact
---------------------

None. This spec is about internal modifications to the code structure.

Developer Impact
----------------

This changes the way the manager objects are constructed.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek

Work Items
----------

- Update objects to expect dependencies passed into the constructor
- Update documentation


Dependencies
============

None.


Documentation Impact
====================

The developer documentation will need to be updated to remove/update code
samples that show ``keystone.common.dependency`` usage. This is primarily
in ``doc/source/extension_development.rst``.


References
==========

Related items:

* `Kilo Summit Etherpad
  <https://etherpad.openstack.org/p/kilo-keystone-object-lifecycle>`_
* `Stevedore`_

.. _Stevedore: http://stevedore.readthedocs.org/en/latest/

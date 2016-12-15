..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Add Multi-Version Support To Federation Mappings
================================================

`bp versioned-mappings <https://blueprints.launchpad.net/keystone/+spec/versioned-mappings>`_


Our current mapping implementation was designed to take data out of a SAML2
assertion and provide a very crude way to use that data to define the user and
groups. We should have a proper way to extend the mapping syntax and not have
to worry about breaking existing mappings. Adding a version to the mapping will
allow us to make changes that are not backward compatible as needed.


Problem Description
===================

Currently there is no way to make changes that are not backward compatible. For
example, it would be nice to fix some of our mapping warts such as:

1. Not all remote rules provided an entry in the data that is used by the local
   rules.  This causes confusion for new users and forces a bit of duplication
   in the remote rules.

In many different conversations with many different people, I've heard the need
to extend mapping. Sometimes the new feature would be backward compatible and
sometimes not. Here is an idea of what I have heard (these things are not going
to be implemented by this spec):

1. Direct mapping from a remote rule into the assertion data. Currently you add
   an entry in the remote section to match something our of the environment and
   make it available via index in the local rule. Why not just have the local
   rule grab the data directly?
2. Allow transformation using something like `Jinja filters`_ to make it
   possible to do some basic changes to the data. For example, (un)urlencoding,
   lower/upper casing, etc.

There are lots more and I could probably talk about it for quite a while. I'm
not going to because I don't know how many of those cases have real customer
use cases and who/when they would be worked on. They are really just evidence
that there are potential changes coming that will only be possible if the
mappings have a version.

.. _Jinja filters: http://jinja.pocoo.org/docs/dev/templates/#filters


Proposed Change
===============

1. Change the federation backend interface for mapping to allow an optional
   version.
2. Change the API to allow an optional version for the mapping.
3. (Optional in this cycle) Provide the code architecture for dispatching to
   different implementations of the mapper based on the version.

Any time a version is not specified we'll use "1" as the version.

Alternatives
------------

1. Add the mapping to the mapping rules themselves. This would no longer need
   any database changes. The caveat is that since our top level JSON rule
   structure is an array we'd either have to make significant changes to make
   it an object or we would have to put version information in every rule in
   the array. Putting a version in every rule in the array means that a single
   rule document could have multiple versions of rules.
2. Do nothing. Do not allow any significant change to the mapping syntax and
   let it die on the vine.

Security Impact
---------------

None. At this point we are not changing the mapping in any way that would cause
changes to how the mappings generate users or groups.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None. We are just enabling future changes that will expand what a deployer
can do with mappings.

Developer Impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dstanek

Work Items
----------

See the `Proposed Change`_ for a detailed list of work items.


Dependencies
============

None.


Documentation Impact
====================

The change won't require anything specific for the docs team. Though, a
"nice to have" change would be to direct people to start explicitly
specifying the version.

Future changes to the mapping engine will require documentation changes, but
those will also have their own specifications.


References
==========

None.

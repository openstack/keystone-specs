..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Replace the concept of extensions
=================================

`bp replace-extensions <https://blueprints.launchpad.net/keystone/+spec/replace-extensions>`_


Replace the existing concept of extensions to Keystone with support for stable,
experimental and out-of-tree functionality.


Problem Description
===================

The approach of on-boarding new functionality via individual, wsgi-pluggable
extensions has been tried in a number of projects. While it does allow a
cloud provider to clearly decide what parts of new functionality they wish to
enable, it also creates a headache for any clients or other service using
Keystone as to what is actually supported in any given installation (at any
particular time). Nova has experienced significant issues with this, having
many such extensions to its API. It is also not clear from our current
extensions model, whether a given extension is stable, experimental, an
optional replacement for some core API etc.

What is required is a more strict definition of the state of new functionality,
while minimizing the confusion this creates in terms of what functionality is
actually available.

Proposed Change
===============

It is proposed that in Keystone abandon the current mechanism of extensions in
favor of three classes of functionality:

* functionality that is stable and part of the Keystone tree
* functionality that is experimental, but still part of the Keystone tree
* functionality that is out-of-tree, can be loaded as required but no
  formal support is provided. Such out-of-tree functionality will typically
  be using the same standard plug points to Keystone as in-tree functionality,
  it is just that this functionality is not part of what we consider core.

The JSON-Home capability would indicate which category any given bit of
functionality was in. A query parameter will be available to get
a specific classification (such as experimental, e.g.
`GET /v3/?experimental=True`).

All existing contrib contents would be re-classified into one of the first two
categories above, with the majority being marked as stable (although we might
possibly also mark some as deprecated). The implication of this is that
all such contrib items would now automatically be loaded and not dependent on
the wsgi pipeline settings.

By default, major new functionality that is proposed to be in-tree will start
off in experimental status. Typically it would take at minimum of one cycle to
transition from experimental to stable, although in special cases this might
happened within a cycle. Experimental status indicates that although the
intention is to keep the API unchanged, we reserve the right to change it up
until the point we mark it as stable. It is not intended that functionality
should stay in experimental for a long period. It should either graduate to
stable or be removed (or perhaps moved to out-of-tree). Functionality that
stays experimental for more than 2 releases, would be expected to make a
transition to one of the other states.

Any new functionality that is proposed as experimental that raises concerns
over security would have to have a config switch to disable it (with disabled
being the default setting). This requirement would be specified as part of the
spec for this new functionality.  A disabled set of functionality should still
respond to a JSON-Home request, but with status of disabled. If the
functionality is disabled, the response should be an HTTP 403 (FORBIDDEN). A
HTTP 404 (NOT FOUND) should not occur from any resource listed in the JSON-Home
document even if it is classified as experimental and disabled.

Alternatives
------------

We keep the current extension mechanism, live with the complexity in the
clients, but perhaps enhance the JSON-Home support to allow better
detection.

Data Model Impact
-----------------

Currently extensions have their own SQL repos - and this wouldn't be
implicitly changed by this proposal. We would likely collapse some of these
into the main repo and prune the upgrade support for the N-2 release.

For new functionality, it would be recommended to use the main repo, in order
to minimize the steps required for administrators given that functionality
is all "part of core".

REST API Impact
---------------

None.

Security Impact
---------------

None, other than the issue mentioned over using config to disable new
functionality that has security concerns.

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

It would no longer be possible to disable individual experimental functionality
by simply removing it from the wsgi pipeline. Rather for those items that are
judged to have a security impact, a config switch would be used.

Developer Impact
----------------

For developers building functionality that is classed as out-of-tree, then they
must provide their own hosted environment for the code as well as a delivery
mechanism. Stackforge is the recommended place to supply such code.

Implementation
==============

Assignee(s)
-----------
* Morgan Fainberg (mdrnstm)
* Steve Martinelli (stevemar)
* Henry Nash (henry-nash)
* Brant Knudson (bknudson)
* Adam Young (ayoung)

Work Items
----------

* Classify current extensions as either stable or experimental
* Update JSON-Home base document to support inclusion of experimental
* Update documentation to indicate expirimental and stable features
* Add documentation to describe graduation process for experimental to stable
* Add documentation to describe removal for non-graduating experimental

Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

Changes to the documentation on extension building and enabling.

References
==========

None

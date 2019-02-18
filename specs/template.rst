..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Example Spec - The title of your blueprint
==========================================

`bug #XXXXXXX <https://bugs.launchpad.net/keystone/+bug/XXXXXXX>`_


Introduction paragraph -- why are we doing anything? A single paragraph of
prose that operators can understand.

Some notes about using this template:

* Your spec should be in `ReSTructured text
  <http://sphinx-doc.org/rest.html>`_, like this template.

* Wrap text at 79 columns.

* The filename in the git repository should be descriptive of the feature.
  For example, if the feature is called "new feature", the file should be
  named new-feature.rst

  Note that this process no longer uses the blueprints in launchpad and
  instead uses a bug for tracking. Bugs are superior to blueprints in that
  patches proposed and merged get a comment and comments on bugs are not
  mutable. This ensures that the tracking of patches in our bug tracker
  cannot be accidently lost if someone edits the body of the bug (unlike
  blueprints, where all data is similar to the bug description and has
  no history tracking.

* You will need to open a bug in launchpad for tracking the changes. This
  bug will be linked in the commit message of each patch for the feature
  (including this specification) so that when a change is made the CI
  system generate a comment on the bug and help us track all work being
  done towards the feature. The bug number should replace the ``XXXXXXX``
  above in the link.

  Please be sure to use ``partial-bug: #XXXXXXX`` in the commit message
  for landing this spec (where ``XXXXXXX`` is the bug id.

* Do not delete any of the sections in this template.  If you have
  nothing to say for a whole section, just write: None

* To test out your formatting, build the docs using tox, or see:
  http://rst.ninjs.org

* If you would like to provide a diagram with your spec, ascii diagrams are
  required.  http://asciiflow.com/ is a very nice tool to assist with making
  ascii diagrams.  The reason for this is that the tool used to review specs is
  based purely on plain text.  Plain text will allow review to proceed without
  having to look at additional files which can not be viewed in gerrit.  It
  will also allow inline feedback on the diagram itself.


Problem Description
===================

A detailed description of the problem:

* For a new feature, this might be use cases. Ensure you are clear about the
  actors in each use case: End User vs Deployer

* For a major reworking of part of the code, it would describe the problems in
  that feature that are being addressed.

This section should clearly communicate *why* it is desirable for the community
to have a solution to the issue.

Proposed Change
===============

Here is where you describe the change you propose to make. How do you propose
to solve this problem? Address the issue at an architectural level, leaving the
implementation details for code review.

If this is one part of a larger effort make it clear where this piece ends. In
other words, what's the scope of this effort?

Alternatives
------------

What other ways could we do this thing? Why aren't we using those? This doesn't
have to be a full literature review, but it should demonstrate that thought has
been put into why the proposed solution is an appropriate one.

Security Impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

* Does this change involve cryptography or hashing?

* Does this change require the use of sudo or any elevated privileges?

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

For more detailed guidance, please see the OpenStack Security Guidelines as
a reference (https://wiki.openstack.org/wiki/Security/Guidelines).  These
guidelines are a work in progress and are designed to help you identify
security best practices.  For further information, feel free to reach out
to the OpenStack Security Group at openstack-security@lists.openstack.org.

Notifications Impact
--------------------

Please specify any changes to notifications. Be that an extra notification,
changes to an existing notification, or removing a notification.

Other End User Impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* Does this change have an impact on python-keystoneclient? What does the user
  interface there look like?

Performance Impact
------------------

Describe any potential performance impact on the system, for example
how often will new code be called, and is there a major change to the calling
pattern of existing code.

Examples of things to consider here include:

* A small change in a utility function or a commonly used decorator can have a
  large impacts on performance.

* Calls which result in a database queries can have a profound impact on
  performance when called in critical sections of the code.

* Will the change include any locking, and if so what considerations are there
  on holding the lock?

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* What config options are being added? Should they be more generic than
  proposed (for example a flag that other hypervisor drivers might want to
  implement as well)? Are the default values ones which will work well in
  real deployments?

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?

* If this change is a new binary, how would it be deployed?

* Please state anything that those doing continuous deployment, or those
  upgrading from the previous release, need to be aware of. Also describe
  any plans to deprecate configuration values or features.  For example, if we
  change the directory name that instances are stored in, how do we handle
  instance directories created before the change landed?  Do we move them?  Do
  we have a special case in the code? Do we assume that the operator will
  recreate all the instances in their cloud?

Developer Impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the driver API, discussion of how
  other backends would implement the feature is required.


Implementation
==============

Assignee(s)
-----------

Who is leading the writing of the code? Or is this a blueprint where you're
throwing it out there to see who picks it up?

If more than one person is working on the implementation, please designate the
primary author and contact.

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* Include specific references to specs and/or blueprints in keystone, or in
  other projects, that this one either depends on or is related to.

* If this requires functionality of another project that is not currently used
  by Keystone (such as the glance v2 API when we previously only required v1),
  document that fact.

* Does this feature require any new library dependencies or code otherwise not
  included in OpenStack? Or does it depend on a specific version of library?


Documentation Impact
====================

What is the impact on the docs team of this change? Some changes might require
donating resources to the docs team to have the documentation updated. Don't
repeat details discussed above, but please reference them here.


References
==========

Please add any useful references here. You are not required to have any
reference. Moreover, this specification should still make sense when your
references are unavailable. Examples of what you could include are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related specifications as appropriate (e.g.  if it's an EC2 thing, link the
  EC2 docs)

* Anything else you feel it is worthwhile to refer to

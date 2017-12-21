..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
Policy in code
==============

`bp policy-in-code <https://blueprints.launchpad.net/keystone/+spec/policy-in-code>`_

Deployers currently have to maintain policy files regardless if they change the
default policy provided from keystone. Maintaining the policy files can be
cumbersome and error-prone. In addition to that we don't use any tooling to
generate or communicate policy changes during the release process. By moving
policy into code, we can leverage tooling to make maintenance easier for
deployers and move to better default policy values.

Problem Description
===================

Today policy exists in a file that deployers are expected to maintain in their
deployment. If a deployer needs to change the default policy rules for an
operation, they have to make those changes and continuously check to make sure
conflicts are resolved with each new release of the policy file. This is
cumbersome to maintain, even if a deployment is only using the default policy.

Proposed Change
===============

The proposed solution is to check policy into the code base and register it
using the oslo.policy library. This is very similar to how projects register
and use configuration options using oslo.config. If policies are provided in a
policy file on disk, those policies will be registered instead of the in-code
default. This provides a way for deployers to override the default we provide.

The registration will need two pieces of data:

1. The operation, e.g. "identity:get_user" or "identity:update_project"
2. The rule, e.g. "role:admin" or "rule:admin_or_owner"

Descriptions can also be provided in the registered policy object that help
describe the operation and the rule or role that is required to execute it.
This description can be used when generating sample policy files from
registered rules.

This is the exact same approach `nova
<http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/policy-in-code.html>`_
used to codify policy.

The following are benefits from the approach:

* There is no longer a need to maintain a policy file in tree.
* We can provide a tool to notify operators when new policies are changed or
  added.
* We can provide a tool to help reduce redundancy in policy files.
* It will be easier to provide a description of each policy much like we do
  configuration options.

Alternatives
------------

An alternative approach was to pull policy into keystone as an official
resource. This would still require some sort of policy override ablility for
deployments that do not wish to deploy the default.

Security Impact
---------------

None, this change only moves where policy is defined and allows for it to be
overridden if necessary.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None. Policy will continue to be evaluated and enforced like it does today.

Performance Impact
------------------

The performance impact of moving policy in code should be minimal. If the
deployment doesn't have a policy file on disk, the service will not have to
fetch it. Instead the default will be registered and used from within code. In
the event the deployment is using policy overrides, the combination of the two
approaches might cause some performance impact compared to defaults in code,
but the overall impact should be negligible.

Other Deployer Impact
---------------------

If a deployer already makes modifications to the default policy file, they
will have to continue maintaining those changes. For deployers who modify a
subset or none of the policy entries, they can essentially remove their policy
file, or the policies that are the default. The end result should be a policy
file that purely consists of overrides the deployer wishes to enforce.

Another deployer impact is that deployers no longer need to double check they
are protecting all new operations by manually inspecting policy files across
releases. Instead, they can be notified about new policies available in a
release and then either assume the well documented default or choose to
override. The current equivalent to this is to compare operations across policy
files without much help from tooling.

Developer Impact
----------------

Any policies added to the code should be registered before they are used. While
the code is switching checks over to context.can() it will be possible to use
policy checks that have not been registered. At some point a hacking check
should be added to disallow the use of oslo_policy.Enforcer.enforce(). There
should also be checks and tests added that make sure new policy entries are
accompanied with a release note.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Anthony Washington (antwash)
  Richard Avelar (ravelar)

Other contributors:
  Lance Bragstad (lbragstad)

Work Items
----------

* Investigate the process for adding oslo.policy into keystone's policy.
* Find place to hook in and register policy.
* Gradually move policy checks from ``policy.json`` into oslo.policy objects.
  This can be done incrementally and should remove the check from
  ``policy.json``.
* Add deployer documentation.
* Remove the policy file from devstack.
* Add sample file generation to write out a merged policy file. This will be
  the effective policy used by keystone, a combination of defaults and
  configured overrides.
* Add a ``keystone-manage`` command to dump a list of policies in a policy file
  which are duplicates of the coded defaults. This will help deployers trim
  policies from existing policy files.


Dependencies
============

None, we should be able to start working on this today.

Documentation Impact
====================

Documentation for deployers about the policy file will be updated to mention
that only policies which differ from the default will need to be included.

References
==========

* `nova specification <http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/policy-in-code.html>`_

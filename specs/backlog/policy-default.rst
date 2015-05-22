..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Default Policy File
==========================================

`bp policy-default <https://blueprints.launchpad.net/keystone/+spec/policy-default>`_

If an endpoint requests a policy file and none is specified for it or for its
service, return a default policy file.


Problem Description
===================

When an endpoint needs to enforce policy, it has to get the policy rules file
from somewhere.  Currently, the OpenStack projects maintain a policy file
in their own git repository.  In order to make it possible for an endpoint to
fetch its policy from Keystone, Keystone needs to have an appropriate policy
file available.  With the specification to merge the existing policy files into
a single, namespaced policy file, there will be an obvious default policy file
to return for services requesting policy from Keystone.  This specification is
https://review.openstack.org/#/c/134656/

Proposed Change
===============

Allow the definition of a default policy file to be returned from the Keystone
policy GET API if there is no policy file that explicitly matches the service
or endpoint.

The Policy rules are all namespaced.  For example, the identity services has
"identity:create_user". This means that multiple services can work off a
single, unified policy file.  If the service requires a rule that does not
match and explicit rule, it will use the default rule.

Alternatives
------------

Keep the policy files split into their component parts by service and require
each service have a default policy file specified.

Security Impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?
*   Yes, policy files used for RBAC

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?
* Yes, it provides a single unified policy file for most operations.

* Does this change involve cryptography or hashing?
* No

* Does this change require the use of sudo or any elevated privileges?
* No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.
* No

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.
* No


Notifications Impact
--------------------

Existing notifications for changes of policy should remain unchanged.


Other End User Impact
---------------------

Aside from the API, are there other ways a user will interact with this
feature?

* Does this change have an impact on python-keystoneclient? What does the user
  interface there look like?

Performance Impact
------------------

This change alone should have no performance impact.  The larger policy files
might lead to longer processing times of RBAC, but the changes should be
immaterial.

Other Deployer Impact
---------------------

Discuss things that will affect how you deploy and configure OpenStack
that have not already been mentioned, such as:

* Configuration:
* A single configuration value for default policy which will take a policy ID.
  If the default policy ID is not set, existing behavior will continue.

* Is this a change that takes immediate effect after its merged, or is it
  something that has to be explicitly enabled?
* Explicitly enabled

* Continuous deployment impact
* minimal

* Upgrades from the previous release
* not require upgrades

* Deprecations
* None

Developer Impact
----------------

Additional workflow is going to be required to maintain the default
configuration file in the face of a growing set of services.


Implementation
==============

Assignee(s)
-----------


Primary assignee:
  Adam Young ayoung ayoung@redhat.com

Other contributors:
 David Chen davechen  <wei.d.chen@intel.com>

Work Items
----------

Should be performed in a single commit
*  Create config option
*  Add code to  endpoint_policy extension to return default policy as needed


Dependencies
============

* Real deployment depends on a viable unified policy file.


Documentation Impact
====================

Configuration value will need to be documented.


References
==========

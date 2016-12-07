..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
PCI-DSS Password Requirements API
=================================

`bp pci-dss-password-requirements-api <https://blueprints.launchpad.net/keystone/+spec/pci-dss-password-requirements-api>`_


Part of the PCI-DSS work included enforcing password strength requirements.
These requirements are kept within keystone, but they should be discoverable so
other services can advertize and leverage them.


Problem Description
===================

Released with Newton was the ability to enforce specific requirements on
password strength for local keystone users. Keystone uses these requirements to
check passwords for new and existing users, which works fine in a stand-alone
model. However, when a user is interfacing with horizon the experience can be
suboptimal for the two following reasons:

1. If a user's password fails the requirements, horizon has no way of
   describing the requirements to the user. Typical password systems will tell
   a user the requirements needed, otherwise the user is left guessing random
   password combinations. This is the most important use case if PCI-DSS is
   going to be used effectively in an OpenStack deployment.
2. Client libraries have no way of knowing the password requirements enforced
   by keystone (including Horizon, OSC, python-keystoneclient bindings, etc).
   The regular expression that enforces the password requirements is only used
   within keystone. By letting other services, like horizon, discover the
   regular expression, they can perform quick password validation before
   issuing the password request to keystone.

Proposed Change
===============

We can make the password requirements and the description of the requirements
discoverable via the API. This can be done by leveraging the existing `domain
configuration API
<http://developer.openstack.org/api-ref/identity/v3/index.html?expanded=show-domain-group-option-configuration-detail#show-domain-group-option-configuration>`_.
The changes necessary would be to make ``keystone.conf [security_compliance]``
a whitelisted section of the domain configuration API.  The change would need
to make the API read-only for those values. The following would be an example
of using the domain configuration API to get the password requirements::

    GET /v3/domains/default/config/security_compliance/password_regex
    GET /v3/domains/default/config/security_compliance/password_regex_description

In order to ensure password strength requirements can be fetched by all users,
it could be isolated into its own API, using its own routers, controllers, etc.

If PCI-DSS password checking isn't
enabled, then a ``404 Not Found`` will be returned, implying there are no
requirements to enforce since it's not in configuration.

Alternatives
------------

One alternative is to create a dedicated API for returning the password
requirements as they are known in the ``keystone.conf``. This would be a much
simpler API, but it wouldn't be leveraging any of the domain configuration API
at all. This might be a useful option depending on how invasive it is to
whitelist PCI-DSS password configuration. If PCI-DSS support is available per
domain in the future, this would have to be reworked.

A second alternative would be to expose the variables to horizon through a
copied version of keystone's configuration file. This option would require
horizon and keystone to have the same requirements at all times. If the two
sources of information are out of sync, user experience will be terrible
because horizon will be advertising the wrong requirements or checking against
outdated requirements.

Security Impact
---------------

If we implement this through the existing domain configuration API, there are
some security concerns around the policy for domain configuration. This change
would require keystone to loosen the policy for domain configuration so that it
can be usable for all users, not just admins. If the policy isn't loosened, the
implementation of the specification using domain configuration would only be
useful to administrators, or users with the admin role. Another security
concern we need to be aware of with this approach is making sure password
requirements can't be modified through the API. Allowing password requirement
changes to an entire domain, or deployment, over the API would be a security
concern if not handled at the deployer or administrator level.

If we isolate returning password requirement information to its own API, we can
build a policy for it that doesn't interfere with existing policy. This would
be an advantage of the first alternative listed above.

Regardless, the API for requesting password requirement information needs to be
protected. If password information isn't protected, it would be easy for a
potential attacker to obtain the password requirements for a deployment and
tailor an attack using those requirements.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

This will make it easier for end users to use horizon to manage their passwords
in a PCI-DSS enabled deployment.

Performance Impact
------------------

Keystone password management APIs might require an additional call to get the
password requirements. This isn't a major performance concern since the call to
get password requirements is extremely lightweight and passwords typically
don't change often.

Other Deployer Impact
---------------------

This change would not require any additional work for deployers. Once a
deployer enables PCI-DSS, the password requirement information can be retrieved
from the API. If the information is changed by the deployer, the new
requirements will be retrievable via the API.

Developer Impact
----------------

This change will require a follow-on specification or patch for horizon to
consume the change.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Lance Bragstad <lbragstad@gmail.com>

Other contributors:
  None

Work Items
----------

* Propose a patch to the domain configuration API to allow looking ups for the
  default configuration file, ``keystone.conf``.
* Propose a patch to whitelist ``keystone.conf [security_compliance]
  password_regex`` and ``keystone.conf [security_compliance]
  password_regex_description`` retrieval using the domain configuration API.
* Update policy to allow fetching of password strength requirements
* Test to make sure password requirements cannot be modified through the domain
  configuration API.
* Add developer information to keystone `PCI-DSS documentation
  <http://docs.openstack.org/developer/keystone/security_compliance.html>`_

Dependencies
============

None.

Documentation Impact
====================

The documentation impact will be relatively minimal, as it's a single API call.

References
==========

None.

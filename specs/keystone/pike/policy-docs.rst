..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============
Add Policy Docs
===============

`bp policy-docs <https://blueprints.launchpad.net/keystone/+spec/policy-docs>`_

Today, operators need to read code in order to completely understand what a
specific policy rule actually does. This results in terrible user experience
for deployers and operators. This specification aims to address this by adding
very detailed descriptions for every policy rule.


Problem Description
===================

Having a detailed understanding of the project's source is a requirement in
order to understand policy. This results in operators having to parse source
code in order to audit default policy rules, grant access to a particular API
to a set of users, or restrict access to a particular API.

Proposed Change
===============

To make sure operators have the information they need in order to make
decisions about policy, we should implement the following:

* All descriptions should use the names of entities described in the `Identity API documentation <http://developer.openstack.org/api-ref/identity/>`_

* We should state the URL of the API the policy rule affects, in the same
  format as it appears in the api-ref, i.e.: DELETE /projects/{project_id}

* We should ensure the docs are rendered well in the generated policy file,
  ensuring all the rules are commented out by default

The following example illustrates what would be represented in the sample
policy file::

    # List users
    #
    # GET /users/
    #
    # "identity:list_users": "rule:admin_required",

    # Show details of a specific user
    #
    # GET /users/{user_id}
    #
    # "identity:get_user": "rule:admin_or_owner",

    # Create a new user
    #
    # POST /users/
    #
    # "identity:create_user": "rule:admin_required",

    # Create a trust relationship between two users
    #
    # POST OS-TRUST/trusts/
    #
    # "identity:create_trust": "user_id:%(trust.trustor_user_id)s",

As an example, we would update the policy definition from::

    policy.RuleDefault(USERS % 'show', RULE_AOO),

To something more like this::

    policy.RuleDefault(
        USERS % 'show',
        RULE_AOO,
        description='Show details of a specific user',
        urls=[{method='GET', path='/users/{user_id}'}]
    )

Alternatives
------------

We could attempt to document all of the information in the policy files like we
have in the `past <https://review.openstack.org/#/c/155919>`_, but that is
harder to maintain and not automated or enforced via code.

Security Impact
---------------

A better understanding of what each rule means can only help operators and
developer get the policy configuration correct.

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

Deployers will now be able to generate a policy file that contains well written
descriptions for each operation.

Developer Impact
----------------

Developers must provide descriptions when implementing new APIs or changing
RBAC for existing ones. We can add hacking check to make sure catching these
gaps is automated.

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

* Add support for oslo.policy to be aware of rule/operation descriptions and
  render them properly
* Add documentation for each policy rule
* Ensure sample policy file render correctly
* Add hacking check to fail if expected description is not provided for policy
  rules
* Ensure all administrator and installation guides reference policy files with
  descriptions rendered

Dependencies
============

* This work requires implementing Policy in Code.
* This work also requires extending the oslo.policy library to include
  descriptions and rendering of policy defaults.


Documentation Impact
====================

By doing this, we will be providing operators with be documentation
out-of-the-box. We need to ensure that all installation guides and
administrator guides reference policy files with rendered descriptions.

References
==========

None

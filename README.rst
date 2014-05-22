=================================
OpenStack Identity Specifications
=================================

This git repository is used to hold approved design specifications for additions
to the Keystone project. Reviews of the specs are done in gerrit, using a
similar workflow to how we review and merge changes to the code itself.

The layout of this repository for Keystone specifications is::

  specs/<release>/

The layout of this repository for Keystone Client is::

  specs/keystoneclient/

When a release of Keystone Client is cut, the implemented blueprints will be
moved to a directory specific to that release::

  specs/keystoneclient/<release>/

You can find an example spec in `specs/template.rst`.

Specifications are proposed for a given release by adding them to the
`specs/<release>` directory and posting it for review.  Not all approved
blueprints will get fully implemented. The implementation status of a blueprint
for a given release can be found by looking at the blueprint in Launchpad::

  http://blueprints.launchpad.net/keystone/<blueprint-name>

Incomplete specifications have to be re-proposed for every release.  The review
may be quick, but even if something was previously approved, it should be
re-reviewed to make sure it still makes sense as written.

Prior to the Juno development cycle, this repository was not used for spec
reviews.  Reviews prior to Juno were completed entirely through Launchpad
blueprints::

  http://blueprints.launchpad.net/keystone

Please note, Launchpad blueprints are still used for tracking the
current status of blueprints. For more information, see::

  https://wiki.openstack.org/wiki/Blueprints

For more information about working with gerrit, see::

  https://wiki.openstack.org/wiki/Gerrit_Workflow

To validate that the specification is syntactically correct (i.e. get more
confidence in the Jenkins result), please execute the following command::

  $ tox

After running ``tox``, the documentation will be available for viewing in HTML
format in the ``doc/build/`` directory.

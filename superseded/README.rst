The 'superseded' directory contains a list of specifications that were approved,
but then superseded by another specification.

The various folders named after release cycles in ``specs/keystone/`` should
reflect the final state of features that were implemented in that cycle.

When the release cycle has ended, move any spec that was superseded to this
folder, and add any reasoning to this file.

* spec: functional-testing-setup.rst

  * reason: decided to use ``keystone/ongoing/functional-testing.rst`` instead.

* spec: online-schema-migration.rst

  * reason: decided to use ``keystone/newton/manage-migration.rst`` instead

* spec: ldap3.rst

  * reason: used the py3 fork of python-ldap (pyldap) instead, see:
    https://blueprints.launchpad.net/keystone/+spec/python3 for details.

* spec: password-totp-plugin.rst

  * reason: decided to use ``keystone/ocata/per-user-auth-plugin-requirements.rst`` instead

* spec: unscoped-catalog.rst

  * reason: core team decided it was no longer valid, spec was abandoned.

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Drop Support for Driver Versioning
==================================

Supporting driver versioning is problematic; our current approach does
not adequately solve the problem and creates a maintenance burden with
minimal justification for this technical debt.

Problem Description
===================

- **Fake out implementation is problematic.**  When we create a wrapper to
  support a legacy driver, we attempt to "fake out" any new methods, so
  that they don’t fail or cause an error. This can be easy enough for
  simple drivers, but considerably more difficult for complicated
  implementations, to the point where we have to change the implementation
  in order to support the legacy driver. Or, when that is not possible,
  resort to raise exception.NotImplemented(). This is problematic and can
  lead to unforeseen bugs caused from the "faked out" code and custom
  implementation.

- **Maintainability.** By supporting legacy drivers, we are forced to
  maintain the legacy driver code, wrappers, as well creating our own
  custom drivers in order to thoroughly test that the older version can
  successfully pass all tests. So we have V8 underscore and all of these
  other dependent objects littered throughout the code base that has to be
  maintained and cleaned up for every release. And this is no small feat.
  For example, the patch for creating a V9 driver for only identity, has
  over 600 lines: https://review.openstack.org/#/c/305315/

- **Minimal justification.** We only support driver versions for one
  version back, which means deployers will have to upgrade their current
  custom drivers soon anyway. And realistically, how many operators are
  upgrading to the latest version without upgrading and testing their
  custom drivers?

  In addition, I sent an email to the operators mailing list regarding
  this. I did not receive any feedback from operators that they were
  utilizing driver versioning; nor that dropping support would
  negatively impact them.

Proposed Change
===============

When a driver contract changes, we clearly document it in the release
notes and documentation. Thus, in order for clients to upgrade, they will
need to:
1. Upgrade their custom drivers to meet the new driver contracts.
2. Thoroughly test their custom drivers against the new code base.

For Newton, we would document this change and remove support for legacy
drivers in Ocata.

Alternatives
------------

Continue with current approach or find a new approach to better support
legacy drivers.

Security Impact
---------------

None

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

Custom drivers would need to meet driver contracts when upgrading.

Developer Impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:

- TBD (volunteers welcome)

Work Items
----------

1. Update driver and developer documentation, removing support for
   driver versioning.

2. Document change in the release notes.

3. Plan refactoring work needed for Ocata.

Dependencies
============

Documentation Impact
====================

Minimal documentation changes will be needed; mostly just removing
text.

References
==========

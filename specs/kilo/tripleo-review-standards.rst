..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
TripleO Review Standards
========================

No launchpad blueprint because this isn't a spec to be implemented in code.

Like many OpenStack projects, TripleO generally has more changes incoming to
the projects than it has core reviewers to review and approve those changes.
Because of this, optimizing reviewer bandwidth is important.  This spec will
propose some changes to our review process discussed at the Paris OpenStack
Summit and intended to make the best possible use of core reviewer time.

There are essentially two major areas that a reviewer looks at when reviewing
a given change: design and implementation.  The design part of the review
covers things like whether the change fits with the overall direction of the
project and whether new code is organized in a reasonable fashion.  The
implementation part of a review will get into smaller details, such as
whether language functionality is being used properly and whether the general
sections of the code identified in the design part of the review do what is
intended.

Generally design is considered first, and then the reviewer will drill down to
the implementation details of the chosen design.

Problem Description
===================
Many times an overall design for a given change will be agreed upon early in
the change's lifecycle.  The implementation for the design may then be
tweaked multiple times (due to rebases, or specific issues pointed out by
reviewers) without any changes to the overall design.  Many times these
implementation details are small changes that shouldn't require much
review effort, but because of our current standard of 2 +2's on the current
patch set before a change can be approved, reviewers often must unnecessarily
revisit a change even when it is clear that everyone involved in the review
is in favor of it.

Proposed Change
===============
When appropriate, allow a core reviewer to approve a change even if the
latest patch set does not have 2 +2's.  Specifically, this should be used
under the following circumstances:

* A change that has had multiple +2's on past patch sets, indicating an
  agreement from the other reviewers that the overall design of the change
  is good.
* Any further alterations to the change since the patch set(s) with +2's should
  be implementation details only - trivial rebases, minor syntax changes, or
  comment/documentation changes.  Any more significant changes invalidate this
  option.

As always, core reviewers should use their judgment.  When in doubt, waiting
for 2 +2's to approve a change is always acceptable, but this new policy is
intended to make it socially acceptable to single approve a change under the
circumstances described above.

When approving a change in this manner, it is preferable to leave a comment
explaining why the change is being approved without 2 +2's.

Alternatives
------------

Allowing a single +2 on "trivial" changes was also discussed, but there were
concerns from a number of people present that such a policy might cause more
trouble than it was worth, particularly since "trivial" changes by nature do
not require much review and therefore don't take up much reviewer time.

Security Impact
---------------

Should be minimal to none.  If a change between patch sets is significant
enough to have a security impact then this policy does not apply.

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Core reviewers will spend less time revisiting patches they have already
voted in favor of, and contributors should find it easier to get their
patches merged because they won't have to wait as long after rebases and
minor changes.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bnemec

Other contributors:
  All cores should review and implement this spec in their reviewing

Work Items
----------

Publish the agreed-upon guidelines somewhere more permanent than a spec.


Dependencies
============

None

Testing
=======

None

Documentation Impact
====================

A new document will need to be created for core reviewers to reference.


References
==========

https://etherpad.openstack.org/p/kilo-tripleo-summit-reviews

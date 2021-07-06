=================
Patch Abandonment
=================

Goal
====

Provide basic policy that core reviewers can apply to outstanding reviews. As
always, it is up to the core reviewers discretion on whether a patch should or
should not be abandoned. This policy is just a baseline with some basic rules.

Problem Description
===================

TripleO consists of many different projects in which many patches become stale
or simply forgotten. This can lead to problems when trying to review the
current patches for a given project.

When to Abandon
===============

If a proposed patch has been marked -1 WIP by the author but has sat idle for
more than 180 days, a core reviewer should abandon the change with a reference
to this policy.

If a proposed patch is submitted and given a -2 and the patch has sat idle for
90 days with no effort to address the -2, a core reviewer should abandon the
change with a reference to this policy.

If a proposed patch becomes stale by ending up with a -1 from CI for 90 days
and no activity to resolve the issues, a core reviewer should abandon the
change with a reference to this policy.

If a proposed patch with no activity for 90 days is in merge conflict, even
with a +1 from CI, a core reviewer should abandon the change with a reference
to this policy.

When NOT to Abandon
===================

If a proposed patch has no feedback but is +1 from CI, a core reviewer should
not abandon such changes.

If a proposed patch a given a -1 by a reviewer but the patch is +1 from CI and
not in merge conflict and the author becomes unresponsive for a few weeks,
reviewers can leave a reminder comment on the review to see if there is
still interest in the patch.  If the issues are trivial then anyone should feel
welcome to checkout the change and resubmit it using the same change ID to
preserve original authorship. Core reviewers should not abandon such changes.

Restoration
===========

Feel free to restore your own patches. If a change has been abandoned
by a core reviewer, anyone can request the restoration of the patch by
asking a core reviewer on IRC in #tripleo on OFTC or by sending a
request to the openstack-dev mailing list. Should the patch again
become stale it may be abandoned again.

Alternative & History
=====================

This topic was previously brought up on the openstack mailing list [1]_ along
with proposed code to use for automated abandonment [2]_. Similar policies are
used by the Puppet OpenStack group [3]_.

Implementation
==============

Author(s)
---------

Primary author:
  aschultz

Other contributors:
  bnemec

Milestones
----------

Pike-2

Work Items
----------

References
==========

.. [1] http://lists.openstack.org/pipermail/openstack-dev/2015-October/076666.html
.. [2] https://github.com/cybertron/tripleo-auto-abandon
.. [3] https://docs.openstack.org/developer/puppet-openstack-guide/reviews.html#abandonment

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Pike
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

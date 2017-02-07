=====================
 Expedited Approvals
=====================

In general, TripleO follows the standard "2 +2" review standard, but there are
situations where we want to make an exception.  This policy is intended to
document those exceptions.

Problem Description
===================

Core reviewer time is precious, and there is never enough of it.  In some
cases, requiring 2 +2's on a patch is a waste of that core time, so we need
to be reasonable about when to make exceptions.  While core reviewers are
always free to use their judgment about when to merge or not merge a patch,
it can be helpful to list some specific situations where it is acceptable and
even expected to approve a patch with a single +2.

Part of this information is already in the wiki, but the future of the wiki
is in doubt and it's better to put policies in a place that they can be
reviewed anyway.

Policy
======

Single +2 Approvals
-------------------

A core can and should approve patches without a second +2 under the following
circumstances:

* The change has multiple +2's on previous patch sets, indicating an agreement
  from the other cores that the overall design is good, and any alterations to
  the patch since those +2's must be minor implementation details only -
  trivial rebases, minor syntax changes, or comment/documentation changes.
* Backports proposed by another core reviewer.  Backports should already have
  been reviewed for design when they merged to master, so if two cores agree
  that the backport is good (one by proposing, the other by reviewing), they
  can be merged with a single +2 review.
* Requirements updates proposed by the bot.
* Translation updates proposed by the bot. (See also `reviewing
  translation imports
  <http://docs.openstack.org/developer/i18n/reviewing-translation-import.html>`_.)

Co-author +2
------------

Co-authors on a patch are allowed to +2 that patch, but at least one +2 from a
core not listed as a co-author is required to merge the patch.  For example, if
core A pushes a patch with cores B and C as a co-authors, core B and core C are
both allowed to +2 that patch, but another core is required to +2 before the
patch can be merged.

Self-Approval
-------------

It is acceptable for a core to self-approve a patch they submitted if it has the
requisite 2 +2's and a CI pass.  However, this should not be done if there is any
dispute about the patch, such as on a change with 2 +2's and an unresolved -1.

Note on CI
----------

This policy does not affect CI requirements.  Patches must still pass CI before
merging.

Alternatives & History
======================

This policy has been in effect for a while now, but not every TripleO core is
aware of it, so it is simply being written down in an official location for
reference.

Implementation
==============

Author(s)
---------

Primary author:
  bnemec

Milestones
----------

The policy is already in effect.

Work Items
----------

Ensure all cores are aware of the policy.  Once the policy has merged, an email
should be sent to openstack-dev referring to it.

References
==========

Existing wiki on review guidlines:
https://wiki.openstack.org/wiki/TripleO/ReviewGuidelines

Previous spec that implemented some of this policy:
http://specs.openstack.org/openstack/tripleo-specs/specs/kilo/tripleo-review-standards.html

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
   * - Newton
     - Added co-author +2 policy
   * - Ocata
     - Added note on translation imports

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

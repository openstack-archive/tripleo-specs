=========================
 Spec Review Process
=========================

Document the existing process to help reviewers, especially newcomers,
understand how to review specs. This is migrating the existing wiki
documentation into a policy.

Problem Description
===================

Care should be taken when approving specs. An approved spec, and an
associated blueprint, indicate that the proposed change has some
priority for the TripleO project. We don't want a bunch of approved
specs sitting out there that no community members are owning or working
on. We also want to make sure that our specs and blueprints are easy to
understand and have sufficient enough detail to effectively communicate
the intent of the change. The more effective the communication, the
more likely we are to elicit meaningful feedback from the wider
community.

Policy
======

To this end, we should be cognizant of the following checklist when
reviewing and approving specs.

* Broad feedback from interested parties.

  * We should do our best to elicit feedback from operators,
    non-TripleO developers, end users, and the wider OpenStack
    community in general.
  * Mail the appropriate lists, such as opentack-operators and
    openstack-dev to ask for feedback. Respond to feedback on the list,
    but also encourage direct comments on the spec itself, as those
    will be easier for other spec reviewers to find.

* Overall consensus

  * Check for a general consensus in the spec.
  * Do reviewers agree this change is meaningful for TripleO?
  * If they don't have a vested interest in the change, are they at
    least not objecting to the change?

* Review older patchsets to make sure everything has been addressed

  * Have any reviewers raised objections in previous patchsets that
    were not addressed?
  * Have any potential pitfalls been pointed out that have not been
    addressed?

* Impact/Security

  * Ensure that the various Impact (end user, deployer, etc) and
    Security sections in the spec have some content.
  * These aren't sections to just gloss off over after understanding
    the implementation and proposed change. They are actually the most
    important sections.
  * It would be nice if that content had elicited some feedback. If it
    didn't, that's probably a good sign that the author and/or
    reviewers have not yet thought about these sections carefully.

* Ease of understandability

  * The spec should be easy to understand for those reviewers who are
    familiar with the project. While the implementation may contain
    technical details that not everyone will grasp, the overall
    proposed change should be able to be understood by folks generally
    familiar with TripleO. Someone who is generally familiar with
    TripleO is likely someone who has run through the undercloud
    install, perhaps contributed some code, or participated in reviews.
  * To aid in comprehension, grammar nits should generally be corrected
    when they have been pointed out. Be aware though that even nits can
    cause disagreements, as folks pointing out nits may be wrong
    themselves. Do not bikeshed over solving disagreements on nits.

* Implementation

  * Does the implementation make sense?
  * Are there alternative implementations, perhaps easier ones, and if
    so, have those been listed in the Alternatives section?
  * Are reasons for discounting the Alternatives listed in the spec?

* Ownership

  * Is the spec author the primary assignee?
  * If not, has the primary assignee reviewed the spec, or at least
    commented that they agree that they are the primary assignee?

* Reviewer workload

  * Specs turn into patches to codebases.
  * A +2 on a spec means that the core reviewer intends to review the
    patches associated with that spec in addition to their other core
    commitments for reviewer workload.
  * A +1 on a spec from a core reviewer indicates that the core
    reviewer is not necessarily committing to review that spec's
    patches.
  * It's fine to +2 even if the spec also relates to other repositories
    and areas of expertise, in addition to the reviewer's own. We
    probably would not want to merge any spec that spanned multiple
    specialties without a representative from each group adding their
    +2.
  * Have any additional (perhaps non-core) reviewers volunteered to
    review patches that implement the spec?
  * There should be a sufficient number of core reviewers who have
    volunteered to go above and beyond their typical reviewer workload
    (indicated by their +2) to review the relevant patches. A
    "sufficient number" is dependent on the individual spec and the
    scope of the change.
  * If reviewers have said they'll be reviewing a spec's patches
    instead of patches they'd review otherwise, that doesn't help much
    and is actually harmful to the overall project.

Alternatives & History
======================

This is migrating the already agreed upon policy from the wiki.

Implementation
==============

Author(s)
---------

Primary author:
  james-slagle (from the wiki history)

Other contributors:
  jpichon

Milestones
----------

None

Work Items
----------

Once the policy has merged, an email should be sent to openstack-dev
referring to this document.

References
==========

* Original documentation: https://wiki.openstack.org/wiki/TripleO/SpecReviews

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ocata
     - Migrated from wiki

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

==================
Tech Debt Tracking
==================

Goal
====

Provide a basic policy for tracking and being able to reference tech debt
related changes in TripleO.

Problem Description
===================

During the development of TripleO, sometimes tech debt is acquired due to time
or resource constraints that may exist. Without a solid way of tracking when
we intentially add tech debt, it is hard to quantify how much tech debt is
being self inflicted. Additionally tech debt gets lost in the code and without
a way to remember where we left it, it is almost impossible to remember when
and where we need to go back to fix some known issues.

Proposed Change
===============

Tracking Code Tech Debt with Bugs
---------------------------------

Intentionally created tech debt items should have a bug [1]_ created with the
`tech-debt` tag added to it. Additionally the commit message of the change
should reference this `tech-debt` bug and if possible a comment should be added
into the code referencing who put it in there.

Example Commit Message::

  Always exit 0 because foo is currently broken

  We need to always exit 0 because the foo process eroneously returns
  42. A bug has been reported upstream but we are not sure when it
  will be addressed.

  Related-Bug: #1234567

Example Comment::

   # TODO(aschultz): We need this because the world is falling apart LP#1234567
   foo || exit 0

Triaging Bugs as Tech Debt
--------------------------

If an end user reports a bug that we know is a tech debt item, the person
triaging the bug should add the `tech-debt` tag to the bug.

Reporting Tech Debt
-------------------

With the `tech-debt` tag on bugs, we should be able to keep a running track
of the bugs we have labeled and should report on this every release milestone
to see trends around how much is being added and when. As part of our triaging
of bugs, we should strive to add net-zero tech-debt bugs each major release if
possible.


Alternatives
------------

We continue to not track any of these things and continue to rely on developers
to remember when they add code and circle back around to fix it themselves or
when other developers find the issue and remove it.

Implementation
==============

Core reviewers should request that any tech debt be appropriately tracked and
feel free to -1 any patches that are adding tech debt without proper
attribution.

Author(s)
---------

Primary author:
  aschultz

Milestones
----------

Queens-1

Work Items
----------

* aschultz to create tech-debt tag in Launchpad.

References
==========

.. [1] https://docs.openstack.org/tripleo-docs/latest/contributor/contributions.html#reporting-bugs

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Queens
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

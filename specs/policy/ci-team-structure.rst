CI Team Structure
=================

Problem Description
-------------------
The soft analysis over the past one to two years is that landing major new
features and function in CI is difficult while being interrupted by a constant
stream of issues.  Each individual is siloed in their own work, feature or
section of the production chain and there is very little time for thoughtful
peer review and collaborative development.

Policy
------

Goals
^^^^^

  * Increase developer focus, decrease distractions, interruptions, and time
    slicing.
  * Encourage collaborative team development.
  * Better and faster code reviews

Team Structure
^^^^^^^^^^^^^^
  * The Ruck
  * The Rover
  * The Sprint Team

The Ruck
^^^^^^^^
One person per week will be on the front lines reporting failures found in CI.
The Ruck & Rover switch roles in the second week of the sprint.

  * Primary focus is to watch CI, report bugs, improve debug documentation.
  * Does not participate in the sprint
  * Attends the meetings where the team needs to be represented
  * Responds to pings on  #oooq / #tripleo regarding CI
  * Reviews and improves documentation
  * Attends meetings for the group where possible
  * For identification, use the irc nick $user|ruck

The Rover
^^^^^^^^^
The primary backup for the Ruck.  The Ruck should be catching all the issues
in CI and passing the issues to the Rover for more in depth analysis or
resolution of the bug.

  * Back up for the Ruck
  * Workload is driven from the tripleo-quickstart bug queue, the Rover is
    not monitoring CI
  * A secondary input for work is identified technical debt defined in the
    Trello board.
  * Attends the sprint meetings, but is not responsible for any sprint work
  * Helps to triage incoming gerrit reviews
  * Responds to pings on irc #oooq / #tripleo
  * If the Ruck is overwhelmed with any of their responsibilities the
    Rover is the primary backup.
  * For identification, use the irc nick $user|rover

The Sprint Team
^^^^^^^^^^^^^^^
The team is defined at the beginning of the sprint based on availability.
Members on the team should be as focused on the sprint epic as possible.
A member of team should spend 80% of their time on sprint goals and 20%
on any other duties like code review or incoming high priority bugs that
the Rover can not manage alone.

  * hand off interruptions to the Ruck and Rover as much as possible
  * focus as a team on the sprint epic
  * collaborate with other members of the sprint team
  * seek out peer review regarding sprint work
  * keep the Trello board updated daily
      * One can point to Trello cards in stand up meetings for status

The Squads
^^^^^^^^^^
The squads operate as a subunit of the sprint team.  Each squad will operate
with the same process and procedures and are managed by the team catalyst.

  * Current Squads
     * CI
        * Responsible for the TripleO CI system ( non-infra ) and build
          verification.
     * Tempest
        * Responsible for tempest development.

Team Leaders
------------

The team catalyst (TC)
^^^^^^^^^^^^^^^^^^^^^^
The member of the team responsible organizing the group. The team will elect or
appoint a team catalyst per release.

  * organize and plan sprint meetings
  * collect status and send status emails

The user advocate (UA)
^^^^^^^^^^^^^^^^^^^^^^
The member of the team responsible for help to prioritize work.  The team will
elect or appoint a user advocate per release.

  * organize and prioritize the Trello board for the sprint planning
  * monitor the board during the sprint.
  * ensure the right work is being done.

The Squads
^^^^^^^^^^
There are two squads on the CI team.

  * tripleo ci
  * tempest development

Each squad has a UA and they share a TC. Both contribute to Ruck and Rover rotations.


Current Leaders for Rocky
^^^^^^^^^^^^^^^^^^^^^^^^^^
  * team catalyst (ci, tempest) - Matt Young
  * user advocate (ci)          - Gabriele Cerami
  * user advocate (tempest)     - Chandan Kumar

Sprint Structure
^^^^^^^^^^^^^^^^
The goal of the sprint is to define a narrow and focused feature called an epic
to work on in a collaborative way.  Work not completed in the sprint will be
added to the technical debt column of Trello.

**Note:** Each sprint needs a clear definition of done that is documented in
the epic used for the sprint.

Sprint Start ( Day 1 ) - 2.5 hours
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  * Sprints are three weeks in length
  * A planning meeting is attended by the entire team including the Ruck and
    Rover
  * Review PTO
  * Review any meetings that need to be covered by the Ruck/Rover
  * The UA will present options for the sprint epic
  * Discuss the epic, lightly breaking each one down
  * Vote on an epic
  * The vote can be done using a doodle form
  * Break down the sprint epic into cards
  * Review each card
      * Each card must have a clear definition of done
      * As a group include as much detail in the card as to provide enough
        information for an engineer with little to no background with the task.


Sprint End ( Day 15 ) - 2.5 hours
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  * Retrospective
      * team members, ruck and rover only
  * Document any technical debt left over from the sprint
  * Ruck / Rover hand off
  * Assign Ruck and Rover positions
  * Sprint demo - when available
  * Office hours on irc

Scrum meetings - 30 Min
^^^^^^^^^^^^^^^^^^^^^^^
  * Planning meeting, video conference
  * Sprint End, video and irc #oooq on freenode
  * 2 live video conference meetings per week
     * sprint stand up
  * Other days, post status to the team's Trello board and/or cards


TripleoO CI Community meeting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  * A community meeting should be held once a week.
  * The meeting should ideally be conveniently scheduled immediately after
    the TripleO community meeting on #tripleo (OFTC)
  * The CI meeting should be announced as part of the TripleO community meeting
    to encourage participation.

Alternatives & History
----------------------

In the past the CI team has worked as individuals or by pairing up for distinct
parts of the CI system and for certain features.  Neither has been
overwhelmingly successful for delivering features on a regular cadence.

Implementation
--------------

Primary author: Wes Hayutin weshayutin at gmail

Other contributors:
  * Ronelle Landy rlandy at redhat
  * Arx Cruz acruz at redhat
  * Sagi Shnaidman at redhat


Milestones
----------

This document is likely to evolve from the feedback discussed in sprint
retrospectives.  An in depth retrospective should be done at the end of each
upstream cycle.


References
----------

Trello
^^^^^^
A Trello board will be used to organize work. The team is expected to keep the
board and their cards updated on a daily basis.

  * https://trello.com/b/U1ITy0cu/tripleo-ci-squad

Dashboards
^^^^^^^^^^
A number of dashboards are used to monitor the CI

  * http://cistatus.tripleo.org/
  * https://dashboards.rdoproject.org/rdo-dev
  * http://zuul-status.tripleo.org/

Team Notes
^^^^^^^^^^

  * https://etherpad.openstack.org/p/tripleo-ci-squad-meeting

Bug Queue
^^^^^^^^^
  * http://tinyurl.com/yag6y9ne


Revision History
----------------

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Rocky
     - April 16 2018

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License. http://creativecommons.org/licenses/by/3.0/legalcode

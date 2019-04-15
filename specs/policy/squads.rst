==============
TripleO Squads
==============

Scaling-up a team is a common challenge in OpenStack.
We always increase the number of projects, with more contributors
and it often implies some changes in the organization.
This policy is intended to document how we will address this challenge in
the TripleO project.

Problem Description
===================

Projects usually start from a single git repository and very often grow to
dozen of repositories, doing different things.  As long as a project gets
some maturity, people who work together on a same topic needs some space
to collaborate the open way.
Currently, TripleO is acting as a single team where everyone meets
on IRC once a week to talk about bugs, CI status, release management.
Also, it happens very often that new contributors have hard time to find
an area of where they could quickly start to contribute.
Time is precious for our developers and we need to find a way to allow
them to keep all focus on their area of work.

Policy
======

The idea of this policy is to create squads of people who work on the
same topic and allow them to keep focus with low amount of external
distractions.

* Anyone would be free to join and leave a squad at will.
  Right now, there is no size limit for a squad as this is something
  experimental. If we realize a squad is too big (more than 10 people),
  we might re-consider the focus of area of the squad.
* Anyone can join one or multiple squads at the same time. Squads will be
  documented in a place anyone can contribute.
* Squads are free to organize themselves a weekly meeting.
* #tripleo remains the official IRC channel.  We won't add more channels.
* Squads will have to choose a representative, who would be a squad liaison
  with TripleO PTL.
* TripleO weekly meeting will still exist, anyone is encouraged to join,
  but topics would stay high level.  Some examples of topics: release
  management; horizontal discussion between squads, CI status, etc.
  The meeting would be a TripleO cross-projects meeting.

We might need to test the idea for at least 1 or 2 months and invest some
time to reflect what is working and what could be improved.

Benefits
--------

* More collaboration is expected between people working on a same topic.
  It will reflect officially what we have nearly done over the last cycles.
* People working on the same area of TripleO would have the possibility
  to do public and open meetings, where anyone would be free to join.
* Newcomers would more easily understand what TripleO project delivers
  since squads would provide a good overview of the work we do.  Also
  it would be an opportunity for people who want to learn on a specific
  area of TripleO to join a new squad and learn from others.
* Open more possibilities like setting up mentoring program for each squad,
  or specific docs to get involved more quickly.

Challenges
----------

* We need to avoid creating silos and keep horizontal collaboration.
  Working on a squad doesn't meen you need to ignore other squads.

Squads
------

The list tends to be dynamic over the cycles, depending on which topics
the team is working on. The list below is subject to change as squads change.

+-------------------------------+----------------------------------------------------------------------------+
| Squad                         | Description                                                                |
+===============================+============================================================================+
| ci                            | Group of people focusing on Continuous Integration tooling and system      |
+-------------------------------+----------------------------------------------------------------------------+
| upgrade                       | Group of people focusing on TripleO upgrades                               |
+-------------------------------+----------------------------------------------------------------------------+
| validations                   | Group of people focusing on TripleO validations tooling                    |
+-------------------------------+----------------------------------------------------------------------------+
| networking                    | Group of people focusing on networking bits in TripleO                     |
+-------------------------------+----------------------------------------------------------------------------+
| integration                   | Group of people focusing on configuration management (eg: services)        |
+-------------------------------+----------------------------------------------------------------------------+
| security                      | Group of people focusing on security                                       |
+-------------------------------+----------------------------------------------------------------------------+
| edge                          | Group of people focusing on Edge/multi-site/multi-cloud                    |
|                               | https://etherpad.openstack.org/p/tripleo-edge-squad-status                 |
+-------------------------------+----------------------------------------------------------------------------+

.. note::

  Note about CI: the squad is about working together on the tooling used
  by OpenStack Infra to test TripleO, though every squad has in charge of
  maintaining the good shape of their tests.


Alternatives & History
======================

One alternative would be to continue that way and keep a single horizontal
team.  As long as we try to welcome in the team and add more projects, we'll
increase the problem severity of scaling-up TripleO project.
The number of people involved and the variety of topics that makes it really difficult to become able to work on everything.

Implementation
==============

Author(s)
---------

Primary author:
  emacchi

Milestones
----------

Ongoing

Work Items
----------

* Work with TripleO developers to document the area of work for every squad.
* Document the output.
* Document squads members.
* Setup Squad meetings if needed.
* For each squad, find a liaison or a squad leader.


.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

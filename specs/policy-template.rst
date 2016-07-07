..
  This template should be in ReSTructured text.  For help with syntax,
  see http://sphinx-doc.org/rest.html

  To test out your formatting, build the docs using tox, or see:
  http://rst.ninjs.org

  The filename in the git repository should match the launchpad URL,
  for example a URL of
  https://blueprints.launchpad.net/oslo?searchtext=awesome-thing should be
  named awesome-thing.rst.

  For specs targeted at a single project, please prefix the first line
  of your commit message with the name of the project.  For example,
  if you're submitting a new feature for oslo.config, your git commit
  message should start something like: "config: My new feature".

  Wrap text at 79 columns.

  Do not delete any of the sections in this template.  If you have
  nothing to say for a whole section, just write: None

  If you would like to provide a diagram with your spec, ascii diagrams are
  required.  http://asciiflow.com/ is a very nice tool to assist with making
  ascii diagrams.  The reason for this is that the tool used to review specs is
  based purely on plain text.  Plain text will allow review to proceed without
  having to look at additional files which can not be viewed in gerrit.  It
  will also allow inline feedback on the diagram itself.

=========================
 The title of the policy
=========================

Introduction paragraph -- why are we doing anything?

Problem Description
===================

A detailed description of the problem.

Policy
======

Here is where you cover the change you propose to make in detail. How do you
propose to solve this problem?

If the policy seeks to modify a process or workflow followed by the
team, explain how and why.

If this is one part of a larger effort make it clear where this piece ends. In
other words, what's the scope of this policy?

Alternatives & History
======================

What other ways could we do this thing? Why aren't we using those? This doesn't
have to be a full literature review, but it should demonstrate that thought has
been put into why the proposed solution is an appropriate one.

If the policy changes over time, summarize the changes here. The exact
details are always available by looking at the git history, but
summarizing them will make it easier for anyone to follow the desired
policy and understand when and why it might have changed.

Implementation
==============

Author(s)
---------

Who is leading the writing of the policy? If more than one person is
working on it, please designate the primary author and contact.

Primary author:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Milestones
----------

When will the policy go into effect?

If there is a built-in deprecation period for the policy, or criteria
that would trigger it no longer being in effect, describe them.

Work Items
----------

List any concrete steps we need to take to implement the policy.

References
==========

Please add any useful references here. You are not required to have
any references. Moreover, this policy should still make sense when
your references are unavailable. Examples of what you could include
are:

* Links to mailing list or IRC discussions

* Links to notes from a summit session

* Links to relevant research, if appropriate

* Related policies as appropriate

* Anything else you feel it is worthwhile to refer to

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * -
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

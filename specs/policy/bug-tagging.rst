========
Bug tags
========

The main TripleO bug tracker is used to keep track of bugs for multiple
projects that are all parts of TripleO. In order to reduce confusion,
we are using a list of approved tags to categorize them.

Problem Description
===================

Given the heavily interconnected nature of the various TripleO
projects, there is a desire to track all the related bugs in a single
bug tracker. However when it is needed, it can be difficult to narrow
down the bugs related to a specific aspect of the project. Launchpad
bug tags can help us here.

Policy
======

The Launchpad official tags list for TripleO contains the following
tags. Keeping them official in Launchpad means the tags will
auto-complete when users start writing them. A bug report can have any
combination of these tags, or none.

Proposing new tags should be done via policy update (proposing a change
to this file). Once such a change is merged, a member of the driver
team will create/delete the tag in Launchpad.

Tags
----

+-------------------------------+----------------------------------------------------------------------------+
| Tag                           | Description                                                                |
+===============================+============================================================================+
| alert                         | For critical bugs requiring immediate attention. Triggers IRC notification |
+-------------------------------+----------------------------------------------------------------------------+
| ci                            | A bug affecting the Continuous Integration system                          |
+-------------------------------+----------------------------------------------------------------------------+
| config-agent                  | A bug affecting os-collect-config, os-refresh-config, os-apply-config      |
+-------------------------------+----------------------------------------------------------------------------+
| documentation                 | A bug that is specific to documentation issues                             |
+-------------------------------+----------------------------------------------------------------------------+
| low-hanging-fruit             | A good starter bug for newcomers                                           |
+-------------------------------+----------------------------------------------------------------------------+
| networking                    | A bug that is specific to networking issues                                |
+-------------------------------+----------------------------------------------------------------------------+
| promotion-blocker             | Bug that is blocking promotion job(s)                                      |
+-------------------------------+----------------------------------------------------------------------------+
| puppet                        | A bug affecting the TripleO Puppet templates                               |
+-------------------------------+----------------------------------------------------------------------------+
| selinux                       | A bug related to SELinux                                                   |
+-------------------------------+----------------------------------------------------------------------------+
| tripleo-common                | A bug affecting tripleo-common                                             |
+-------------------------------+----------------------------------------------------------------------------+
| tripleo-heat-templates        | A bug affecting the TripleO Heat Templates                                 |
+-------------------------------+----------------------------------------------------------------------------+
| tripleoclient                 | A bug affecting python-tripleoclient                                       |
+-------------------------------+----------------------------------------------------------------------------+
| ui                            | A bug affecting the TripleO UI                                             |
+-------------------------------+----------------------------------------------------------------------------+
| upgrade                       | A bug affecting upgrades                                                   |
+-------------------------------+----------------------------------------------------------------------------+
| validations                   | A bug affecting the Validations                                            |
+-------------------------------+----------------------------------------------------------------------------+
| workflows                     | A bug affecting the Mistral workflows                                      |
+-------------------------------+----------------------------------------------------------------------------+
| xxx-backport-potential        | Cherry-pick request for the stable team                                    |
+-------------------------------+----------------------------------------------------------------------------+

Alternatives & History
======================

The current ad-hoc system is not working well, as people use
inconsistent subject tags and other markers. Likewise, with the list
not being official Launchpad tags do not autocomplete and quickly
become inconsistent, hence not as useful.

We could use the wiki to keep track of the tags, but the future of the
wiki is in doubt. By making tags an official policy, changes to the
list can be reviewed.

Implementation
==============

Author(s)
---------

Primary author:
  jpichon

Milestones
----------

Newton-3

Work Items
----------

Once the policy has merged, someone with the appropriate Launchpad
permissions should create the tags and an email should be sent to
openstack-dev referring to this policy.

References
==========

Launchpad page to manage the tag list:
https://bugs.launchpad.net/tripleo/+manage-official-tags

Thread that led to the creation of this policy:
http://lists.openstack.org/pipermail/openstack-dev/2016-July/099444.html

Revision History
================

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced

.. note::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

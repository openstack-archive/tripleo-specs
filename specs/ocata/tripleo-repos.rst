..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
 TripleO Repo Management Tool
==============================

https://blueprints.launchpad.net/tripleo/tripleo-repos

Create a tool to handle the repo setup for TripleO

Problem Description
===================

The documented repo setup steps for TripleO are currently:

* 3 curls
* a sed
* a multi-line bash command
* a yum install
* (optional) another yum install and sed command

These steps are also implemented in multiple other places, which means every
time a change needs to be made it has to be done in at least three different
places. The stable branches also need slightly different commands which further
complicates the documentation.  They also need to appear in multiple places
in the docs (e.g. virt system setup, undercloud install, image build,
undercloud upgrade).

Proposed Change
===============

Overview
--------

My proposal is to abstract away the repo management steps into a standalone
tool.  This would essentially change the repo setup from the process
described above to something like::

    sudo yum install -y http://tripleo.org/tripleo-repos.rpm
    sudo tripleo-repos current

Historical note: The original proposal was called dlrn-repo because it was
dealing exclusively with dlrn repos.  Now that we've started to add more
repos like Ceph that are not from dlrn, that name doesn't really make sense.

This will mean that when repo setup changes are needed (which happen
periodically), they only need to be made in one place and will apply to both
developer and user environments.

Alternatives
------------

Use tripleo.sh's repo setup.  However, tripleo.sh is not intended as a
user-facing tool.  It's supposed to be a thin wrapper that essentially
implements the documented deployment commands.

Security Impact
---------------

The tool would need to make changes to the system's repo setup and install
packages.  This is the same thing done by the documented commands today.

Other End User Impact
---------------------

This would be a new user-facing CLI.

Performance Impact
------------------

No meaningful change

Other Deployer Impact
---------------------

Deployers would need to switch to this new method of configuring the
TripleO repos in their deployments.

Developer Impact
----------------

There should be little to no developer impact because they are mostly using
other tools to set up their repos, and those tools should be converted to use
the new tool.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bnemec

Other contributors:
  <launchpad-id or None>

Work Items
----------

* Update the proposed tool to match the current repo setup
* Import code into gerrit
* Package tool
* Publish the package somewhere easily accessible
* Update docs to use tool
* Convert existing developer tools to use this tool


Dependencies
============

NA

Testing
=======

tripleo.sh would be converted to use this tool so it would be covered by
existing CI.


Documentation Impact
====================

Documentation would be simplified.


References
==========

Original proposal:
http://lists.openstack.org/pipermail/openstack-dev/2016-June/097221.html

Current version of the tool:
https://github.com/cybertron/dlrn-repo

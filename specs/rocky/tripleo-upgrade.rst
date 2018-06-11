..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
A unified tool for upgrading TripleO based deploments
=====================================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-upgrade

In order to avoid work duplication and automation code being out of sync with the
official documentation we would like to create a single repository hosting the upgrade
automation code that can be run on top of deployments done with various tools.

Problem Description
===================
Currently automation code for TripleO upgrades is spread across several repositories
and it is tightly coupled with the framework being used for deployment, e.g. tripleo-
quickstart or Infrared.

Proposed Change
===============

Overview
--------

Our proposal is to decouple the upgrade automation code and make it deployment tool
agnostic. This way it could be consumed in different scenarios such as CI, automated
or manual testing.

Alternatives
------------

For the previous releases the automation code has been hosted in diffrent repositories
such as tripleo-quickstart-extras, infrared or private repos. This is not convenient
as they all cover basically the same workflow so we are duplicating work. We would like
to avoid this and collaborate on a single repository.

Security Impact
---------------

None.

Other End User Impact
---------------------

This tool allows the users to run the TripleO upgrade in an automated fashion or
semi-automatic by creating scripts for each upgrade step which can be later run manually
by the user.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

This tools helps developers by providing a quick way to run TripleO upgrades. This could
be useful when reproducing and debugging reported issues.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  matbu, mcornea

Work Items
----------

* Create new repository in Openstack Git
* Migrate repository with its history from https://github.com/redhat-openstack/tripleo-upgrade

Dependencies
============

* ansible

Testing
=======


Documentation Impact
====================


References
==========


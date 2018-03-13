..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Support Vitrage(Root Cause Analysis, RCA) Service
==================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/tripleo-vitrage-integration

[Vitrage]_ is the official OpenStack RCA project. It can perfectly organizes,
analyzes and visualizes the holistic view of the Cloud.

Vitrage provides functions as follows:

* A clear view of the Cloud Topology

* Deduced alarms and states

* RCA for alarms/events

Via Vitrage, the end users can understand what happened in a complex cloud
environment, get the root cause of problems and then resolve issues in time.

Problem Description
===================

Currently the installation and configuration of Vitrage in openstack is done
manually or using devstack. It shall be automated via tripleo.

Integration Vitrage in TripleO is benefit for building the OPNFV platform.
It helps the OPNFV [Doctor]_ project using Vitrage as inspector component to
build the fault management framework with [Apex]_ installer which is an OPNFV
installation and deployment tool based on TripleO.

Proposed Change
===============

Overview
--------

This spec proposes changes to automate the deployment of Vitrage using TripleO.

* Add puppet-vitrage package to overcloud-full image.

* Define Vitrage Service in THT.

* Add how and when to deploy Vitrage in puppet-tripleo.

Alternatives
------------

None

Security Impact
---------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

None

Other Deployer Impact
---------------------

Vitrage service is default disabled in a Deployment. Need to enable it
if deployer want to use it.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dong wenjuan <dong.wenjuan@zte.com.cn>

Work Items
----------

As outlined in the proposed changes.

Dependencies
============

The Vitrage RPM package must be in RDO repo.

Testing
=======

Add the test for CI scenarios.

Documentation Impact
====================

The setup and configuration of the Vitrage server should be documented.

References
==========

.. [Vitrage] https://wiki.openstack.org/wiki/Vitrage
.. [Apex] https://wiki.opnfv.org/display/apex
.. [Doctor] https://wiki.opnfv.org/display/doctor

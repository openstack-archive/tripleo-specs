..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
Support Barometer(Software Fastpath Service Quality Metrics) Service
====================================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+spec/tripleo-barometer-integration

The scope of the [Barometer]_ project is to provide interfaces to support
monitoring of the NFVI. The project has plugins for telemetry frameworks
to enable the collection of platform stats and events and relay gathered
information to fault management applications or the VIM. The scope is
limited to collecting/gathering the events and stats and relaying them
to a relevant endpoint.

The consumption of performance and traffic-related information/events
provided by this project should be a logical extension of any existing
VNF/NFVI monitoring framework.

Problem Description
===================

Integration of Barometer in TripleO is a benefit for building the OPNFV platform.
The Barometer project is complementary to the Doctor project to build the fault
management framework with [Apex_Installer]_ installer which is an OPNFV installation and
deployment tool based on TripleO.

Proposed Change
===============

Overview
--------

This spec proposes changes to automate the deployment of Barometer using TripleO.

* Add puppet-barometer package to the overcloud-full image.

* Define Barometer Service in THT.

* Add how and when to deploy Barometer in puppet-tripleo.

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

Barometer service is default disabled in a Deployment. Need to enable it
if deployer wants to use it.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Akhila Kishore <akhila.kishore@intel.com>

Work Items
----------

As outlined in the proposed changes.

Dependencies
============

The Barometer RPM package must be in RDO repo.

Testing
=======

Add the test for CI scenarios.

Documentation Impact
====================

The setup and configuration of the Barometer service should be documented.

References
==========

.. [Barometer] https://wiki.opnfv.org/display/fastpath/Barometer+Home
.. [Apex_Installer] https://wiki.opnfv.org/display/apex

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enable deployment of performace monitoring
==========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-opstools-performance-monitoring

TripleO should have a possibility to automatically setup and install
the performance monitoring agent (collectd) to service the overcloud.

Problem Description
===================

We need to easily enable operators to connect overcloud nodes to performance
monitoring stack. The possible way to do so is to install collectd agent
together with set of plugins, depending on a metrics we want to collect
from overcloud nodes.

Summary of use cases:

1. collectd deployed on each overcloud node reporting configured metrics
(via collectd plugins) to external collector.

Proposed Change
===============

Overview
--------

The collectd service will be deployed as a composable service on
the overcloud stack when it is explicitly stated via environment file.

Security Impact
---------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

Metric collection and transport to the monitoring node can create I/O which
might have performance impact on monitored nodes.

Other Deployer Impact
---------------------

None

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Lars Kellogg-Stedman (larsks)

Other contributors:
  Martin Magr (mmagr)

Work Items
----------

* puppet-tripleo profile for collectd service
* tripleo-heat-templates composable service for collectd deployment

Dependencies
============

* Puppet module for collectd service: puppet-collectd [1]
* CentOS Opstools SIG repo [2]

Testing
=======

We should consider creating CI job for deploying overcloud with monitoring
node to perform functional testing.


Documentation Impact
====================

New template parameters will have to be documented.


References
==========

[1] https://github.com/voxpupuli/puppet-collectd
[2] https://wiki.centos.org/SpecialInterestGroup/OpsTools

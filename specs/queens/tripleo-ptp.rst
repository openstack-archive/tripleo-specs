..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
TripleO PTP (Precision Time Protocol) Support
=============================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ptp

This spec introduces support for a time synchronization method called PTP [0]
which provides better time accuracy than NTP in general. With hardware
timestamping support on the host, PTP can achieve clock accuracy in the
sub-microsecond range, making it suitable for measurement and control systems.

Problem Description
===================

Currently tripleo deploys NTP services by default which provide millisecond
level time accuracy, but this is not enough for some cases:

* Fault/Error events will include timestamps placed on the associated event
  messages, retrieved by detectors with the purpose of accurately identifying
  the time that the event occurred. Given that the target Fault Management
  cycle timelines are in tens of milliseconds on most critical faults, events
  ordering may reverse against actual time if precison and accuracy of clock
  synchronization are in the same level of accuracy.

* NFV C-RAN (Cloud Radio Access Network) is looking for better time
  sychronization and distribution in micro-second level accuracy as alternative
  for NTP, PTP has been evaluated as one of the technologies.

This spec is not intended to cover all the possible ways of PTP usage, rather
to provide a basic deployment path for PTP in tripleo with default
configuration set to support PTP Ordinary Clock (slave mode); the master mode
ptp clock configuration is not in the scope of this spec, but shall be deployed
by user to provide the time source for the PTP Ordinary Clock. The full support
of PTP capability can be enhanced further based on this spec.

User shall be aware of the fact that NTP and PTP can not be configured together
on the same node without a coordinator program like timemaster which is also
provided by linuxptp package. How to configure and use timemaster is not in the
scope of this spec.

Proposed Change
===============

Overview
--------

Provide the capability to configure PTP as time synchronization method:

* Add PTP configuration file path in overcloud resource registry.

* Add puppet-tripleo profile for PTP services.

* Add tripleo-heat-templates composable service for PTP.

Retain the current default behavior to deploy NTP as time synchronization
source:

* The NTP services remain unchanged as the default time synchronization method.

* The NTP services must be disabled on nodes where PTP are deployed.

Alternatives
------------

The alternative is to continue to use NTP.

Security Impact
---------------

Security issues originated from PTP will need to be considered.

Other End User Impact
---------------------

Users will get more accurate time from PTP.

Performance Impact
------------------

No impact with default deployment mode which uses NTP as time source.

Other Deployer Impact
---------------------

The operator who wants to use PTP should identify and provide the PTP capable
network interface name and make sure NTP is not deployed on the nodes where PTP
will be deployed. The default PTP network interface name is set to 'nic1' where
user should change it according to real interface name. By default, PTP will
not be deployed unless explicitly configured.

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zshi

Work Items
----------

* Puppet-tripleo profile for PTP services
* Tripleo-heat-templates composable service for PTP deployment

Dependencies
============

* Puppet module for PTP services: ptp [1]
* The linuxptp RPM must be installed, and PTP capable NIC must be identified.
* Refer to linuxptp project page [2] for the list of drivers that support the
  PHC (Physical Hardware Clock) subsystem.

Testing
=======

The deployment of PTP should be testable in CI.

Documentation Impact
====================

The deployment documation will need to be updated to cover the configuration of
PTP.

References
==========

* [0] https://standards.ieee.org/findstds/standard/1588-2008.html
* [1] https://github.com/redhat-nfvpe/ptp
* [2] http://linuxptp.sourceforge.net

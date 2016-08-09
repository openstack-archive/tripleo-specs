..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Add Adapter Teaming to os-net-config
====================================

https://blueprints.launchpad.net/os-net-config/+spec/os-net-config-teaming

This spec describes adding features to os-net-config to support adapter teaming
as an option for bonded interfaces. Adapter teaming allows additional features
over regular bonding, due to the use of the teaming agent.

Problem Description
===================

os-net-config supports both OVS bonding and Linux kernel bonding, but some
users want to use adapter teaming instead of bonding. Adapter teaming provides
additional options that bonds don't support, and do support almost all of the
options that are supported by bonds.

Proposed Change
===============

Overview
--------

Add a new class similar to the existing bond classes that allows for the
configuration of the teamd daemon through teamdctl. The syntax for the
configuration of the teams should be functionally similar to configuring
bonds.

Alternatives
------------

We already have two bonding methods in use, the Linux bonding kernel module,
and Open vSwitch. However, adapter teaming is becoming a best practice, and
this change will open up that possiblity.

Security Impact
---------------

The end result of using teaming instead of other modes of bonding should be
the same from a security standpoint. Adapter teaming does not interfere with
iptables or selinux.


Other End User Impact
---------------------

Operators who are troubleshooting a deployment where teaming is used may need
to familiarize themselves with the teamdctl utility.

Performance Impact
------------------

Using teaming rather than bonding will have a mostly positive impact on
performance. Teaming is very lightweight, and may use less CPU than other
bonding modes, especially OVS. Teaming has the following impacts:

* Fine-grained control over load balancing hashing algorithms.

* Port-priorities and stickyness

* Per-port monitoring.

Other Deployer Impact
---------------------

In TripleO, os-net-config has existing sample templates for OVS-mode
bonds and Linux bonds. There has been some discussion with Dan Prince
about unifying the bonding templates in the future.

The type of bond could be set as a parameter in the NIC config
templates. To this end, it probably makes sense to make the teaming
configuration as similar to the bonding configurations as possible.

Developer Impact
----------------

If possible, the configuration should be as similar to the bonding
configuration as possible. In fact, it might be treated as a different
form of bond, as long as the required metadata for teaming can be
provided in the options.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Dan Sneddon <dsneddon@redhat.com>

Work Items
----------

* Add teaming object and unit tests.

* Configure sample templates to demonstrate usage of teaming.

* Test TripleO with new version of os-net-config and adapter teaming configured.

Configuration Example
---------------------

The following is an example of a teaming configuration that os-net-config
should be able to implement::

  -
    type: linux_team
    name: team0
    bonding_options: "{"runner": {"name": "activebackup"}, "link_watch": {"name": "ethtool"}}"
    addresses:
      -
        ip_subnet: 192.168.0.10/24
    members:
      -
        type: interface
        name: eno2
        primary: true
      -
        type: interface
        name: eno3

The only difference between a Linux bond configuration and an adapter team
configuration in the above example is the type (linux_team), and the content
of the bonding_options (bonding has a different format for options).

Implementation Details
----------------------

os-net-config will have to configure the ifcfg files for the team. The ifcfg
format for team interfaces is documented here [1].

If an interface is marked as primary, then the ifcfg file for that interface
should list it at a higher than default (0) priority::

  TEAM_PORT_CONFIG='{"prio": 100}'

The mode is set in the runner: statement, as well as any settings that
apply to that teaming mode.

We have the option of using strictly ifcfg files or using the ip utility
to influence the settings of the adapter team. It appears from the teaming
documentation that either approach will work.

The proposed implementation [2] of adapter teaming for os-net-config uses
only ifcfg files to set the team settings, slave interfaces, and to
set the primary interface. The potential downside of this path is that
the interface must be shut down and restarted when config changes are
made, but that is consistent with the other device types in os-net-config.
This is probably acceptable, since network changes are made rarely and
are assumed to be disruptive to the host being reconfigured.

Dependencies
============

* teamd daemon and teamdctl command-line utility must be installed. teamd is
  not installed by default on RHEL/CENTOS, however, teamd is currently
  included in the RDO overcloud-full image. This should be added ot the list
  of os-net-config RPM dependencies.

* For LACP bonds using 802.3ad, switch support will need to be configured and
  at least two ports must be configured for LACP bonding.


Testing
=======

In order to test this in CI, we would need to have an environment where we
have multiple physical NICs. Adapter teaming supports modes other than LACP,
so we could possibly get away with multiple links without any special
configuration.


Documentation Impact
====================

The deployment documentaiton will need to be updated to cover the use of
teaming. The os-net-config sample configurations will demostrate the use
in os-net-config. TripleO Heat template examples should also help with
deployments using teaming.


References
==========

* [1] - Documentation: Creating a Network Team Using ifcfg Files
  https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/sec-Configure_a_Network_Team_Using-the_Command_Line.html#sec-Creating_a_Network_Team_Using_ifcfg_Files

* [2] - Review: Add adapter teaming support using teamd for ifcfg-systems
  https://review.openstack.org/#/c/339854/
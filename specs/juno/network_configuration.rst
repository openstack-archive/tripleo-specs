..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
TripleO network configuration
==========================================

https://blueprints.launchpad.net/tripleo/+spec/os-net-config

We need a tool (or tools) to help configure host level networking
in TripleO. This includes things like:

 * Static IPs

 * Multiple OVS bridges

 * Bonding

 * VLANs

Problem Description
===================

Today in TripleO we bootstrap nodes using DHCP so they can download
custom per node metadata from Heat. This metadata contains per instance
network information that allows us to create a customized host level network
configuration.

Today this is accomplished via two scripts:

 * ensure-bridge: http://git.openstack.org/cgit/openstack/tripleo-image-elements/tree/elements/network-utils/bin/ensure-bridge
 * init-neutron-ovs: http://git.openstack.org/cgit/openstack/tripleo-image-elements/tree/elements/neutron-openvswitch-agent/bin/init-neutron-ovs

The problem with the existing scripts is that their feature set is extremely
prescriptive and limited. Today we only support bridging a single NIC
onto an OVS bridge, VLAN support is limited and more advanced configuration
(of even common IP address attributes like MTUs, etc) is not possible.

Furthermore we also desire some level of control over how networking changes
are made and whether they are persistent. In this regard a provider layer
would be useful so that users can choose between using for example:

 * ifcfg/eni scripts: used where persistence is required and we want
   to configure interfaces using the distro supported defaults
 * iproute2: used to provide optimized/streamlined network configuration
   which may or may not also include persistence

Our capabilities are currently limited to the extent that we are unable
to fully provision our TripleO CI overclouds without making manual
changes and/or hacks to images themselves. As such we need to
expand our host level network capabilities.

Proposed Change
===============

Create a new python project which encapsulates host level network configuration.

This will likely consist of:

 * an internal python library to facilitate host level network configuration

 * a binary which processes a YAML (or JSON) format and makes the associated
   python library calls to configure host level networking.

By following this design the tool should work well with Heat driven
metadata and provide us the future option of moving some of the
library code into Oslo (oslo.network?) or perhaps Neutron itself.

The tool will support a "provider" layer such that multiple implementations
can drive the host level network configuration (iproute2, ifcfg, eni).
This is important because as new network config formats are adopted
by distributions we may want to gradually start making use of them
(thinking ahead to systemd.network for example).

The tool will also need to be extensible such that we can add new
configuration options over time. We may for example want to add
more advanced bondings options at a later point in time... and
this should be as easy as possible.

The focus of the tool initially will be host level network configuration
for existing TripleO features (interfaces, bridges, vlans) in a much
more flexible manner. While we support these things today in a prescriptive
manner the new tool will immediately support multiple bridges, interfaces,
and vlans that can be created in an ad-hoc manner. Heat templates can be
created to drive common configurations and people can customize those
as needed for more advanced networking setups.

The initial implementation will focus on persistent configuration formats
for ifcfg and eni, like we do today via ensure-bridge. This will help us
continue to make steps towards bringing bare metal machines back online
after a power outage (providing a static IP for the DHCP server for example).

The primary focus of this tool should always be host level network
configuration and fine tuning that we can't easily do within Neutron itself.
Over time the scope and concept of the tool may shift as Neutron features are
added and/or subtracted.


Alternatives
------------

One alternative is to keep expanding ensure-bridge and init-neutron-ovs
which would require a significant number of new bash options and arguments to
configure all the new features (vlans, bonds, etc.).

Many of the deployment projects within the OpenStack ecosystem are doing
similar sorts of networking today. Consider:

 * Chef/Crowbar: https://github.com/opencrowbar/core/blob/master/chef/cookbooks/network/recipes/default.rb
 * Fuel: https://github.com/stackforge/fuel-library/tree/master/deployment/puppet/l23network
 * VDSM (GPL): contains code to configure interfaces, both ifcfg and iproute2 abstractions (git clone http://gerrit.ovirt.org/p/vdsm.git, then look at vdsm/vdsm/network/configurators)
 * Netconf: heavy handed for this perhaps but interesting (OpenDaylight, etc)

Most of these options are undesirable because they would add a significant
number of dependencies to TripleO.

Security Impact
---------------

The configuration data used by this tool is already admin-oriented in
nature and will continue to be provided by Heat. As such there should
be no user facing security concerns with regards to access to the
configuration data that aren't already present.

This implementation will directly impact the low level network connectivity
in all layers of TripleO including the seed, undercloud, and overcloud
networks. Any of the host level networking that isn't already provided
by Neutron is likely affected.

Other End User Impact
---------------------

This feature enables deployers to build out more advanced undercloud and
overcloud networks and as such should help improve the reliability and
performance of the fundamental host network capabilities in TripleO.

End users should benefit from these efforts.

Performance Impact
------------------

This feature will allow us to build better/more advanced networks and as
such should help improve performance. In particular the interface bonding
and VLAN support should help in this regard.

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
  Dan Prince (dan-prince on Launchpad)

Work Items
----------

 * Create project on GitHub: os-net-config

 * Import project into openstack-infra, get unit tests gating, etc.

 * Build a python library to configure host level networking with
   an initial focus on parity with what we already have including things
   we absolutely need for our TripleO CI overcloud networks.

   The library will consist of an object model which will allow users to
   create interfaces, bridges, and vlans, and bonds (optional). Each of
   these types will act as a container for address objects (IPv4 and IPv6)
   and routes (multiple routes may be defined). Additionally, each
   object will include options to enable/disable DHCP and set the MTU.

 * Create provider layers for ifcfg/eni. The providers take an object
   model and apply it ("make it so"). The ifcfg provider will write out
   persistent config files in /etc/sysconfig/network-scripts/ifcfg-<name>
   and use ifup/ifdown to start and stop the interfaces when an change
   has been made. The eni provider will write out configurations to
   /etc/network/interfaces and likewise use ifup/ifdown to start and
   stop interfaces when a change has been made.

 * Create a provider layer for iproute2. Optional, can be done at
   a later time. This provider will most likely not use persistent
   formats and will run various ip/vconfig/route commands to
   configure host level networking for a given object model.

 * Create a binary that processes a YAML config file format and makes
   the correct python library calls. The binary should be idempotent
   in that running the binary once with a given configuration should
   "make it so". Running it a second time with the same configuration
   should do nothing (i.e. it is safe to run multiple times). An example
   YAML configuration format is listed below which describes a single
   OVS bridge with an attached interface, this would match what
   ensure-bridge creates today:

.. code-block:: yaml

  network_config:
    - 
      type: ovs_bridge
      name: br-ctlplane
      use_dhcp: true
      ovs_extra:
        - br-set-external-id br-ctlplane bridge-id br-ctlplane
      members:
        - 
          type: interface
          name: em1

..

   The above format uses a nested approach to define an interface
   attached to a bridge.

 * TripleO element to install os-net-config. Most likely using
   pip (but we may use git initially until it is released).

 * Wire this up to TripleO...get it all working together using the
   existing Heat metadata formats. This would include any documentation
   changes to tripleo-incubator, deprecating old elements, etc.

 * TripleO heat template changes to use the new YAML/JSON formats. Our default
   configuration would most likely do exactly what we do today (OVS bridge
   with a single attached interface). We may want to create some other example
   heat templates which can be used in other environments (multi-bridge
   setups like we use for our CI overclouds for example).


Dependencies
============

None

Testing
=======

Existing TripleO CI will help ensure that as we implement this we maintain
parity with the current feature set.

The ability to provision and make use of our Triple CI clouds without
custom modifications/hacks will also be a proving ground for much of
the work here.

Additional manual testing may be required for some of the more advanced
modes of operation (bonding, VLANs, etc.)

Documentation Impact
====================

The recommended heat metadata used for network configuration may
change as result of this feature. Older formats will be preserved for
backwards compatibility.

References
==========

Notes from the Atlanta summit session on this topic can be found
here (includes possible YAML config formats):

 * https://etherpad.openstack.org/p/tripleo-network-configuration

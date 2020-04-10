..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
TripleO Routed Networks Deployment (Spine-and-Leaf Clos)
========================================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-routed-networks-deployment

TripleO uses shared L2 networks today, so each node is attached to the
provisioning network, and any other networks are also shared. This
significantly reduces the complexity required to deploy on bare metal,
since DHCP and PXE booting are simply done over a shared broadcast domain.
This also makes the network switch configuration easy, since there is only
a need to configure VLANs and ports, but no added complexity from dynamic
routing between all switches.

This design has limitations, however, and becomes unwieldy beyond a certain
scale. As the number of nodes increases, the background traffic from Broadcast,
Unicast, and Multicast (BUM) traffic also increases. This design also requires
all top-of-rack switches to trunk the VLANs back to the core switches, which
centralizes the layer 3 gateway, usually on a single core switch. That creates
a bottleneck which is not present in Clos architecture.

This spec serves as a detailed description of the overall problem set, and
applies to the master blueprint. The sub-blueprints for the various
implementation items also have their own associated spec.

Problem Description
===================

Where possible, modern high-performance datacenter networks typically use
routed networking to increase scalability and reduce failure domains. Using
routed networks makes it possible to optimize a Clos (also known as
"spine-and-leaf") architecture for scalability::

  ,=========.                        ,=========.
  | spine 1 |__                    __| spine 2 |
  '==|\=====\_ \__________________/ _/=====/|=='
     | \_     \___   /       \  ___/     _/ |   ^
     |    \___ /    \ _______ /   \ ___/    |   |-- Dynamic routing (BGP, OSPF,
     |    /  \       /       \      /  \    |   v   EIGRP)
  ,------.    ,------       ,------.    ,------.
  |leaf 1|....|leaf 2|      |leaf 3|....|leaf 4| ======== Layer 2/3 boundary
  '------'    '------'      '------'    '------'
     |            |             |            |
     |            |             |            |
     |-[serv-A1]=-|             |-[serv-B1]=-|
     |-[serv-A2]=-|             |-[serv-B2]=-|
     |-[serv-A3]=-|             |-[serv-B3]=-|
         Rack A                     Rack B



In the above diagram, each server is connected via an Ethernet bond to both
top-of-rack leaf switches, which are clustered and configured as a virtual
switch chassis. Each leaf switch is attached to each spine switch. Within each
rack, all servers share a layer 2 domain. The subnets are local to the rack,
and the default gateway is the top-of-rack virtual switch pair. Dynamic routing
between the leaf switches and the spine switches permits East-West traffic
between the racks.

This is just one example of a routed network architecture. The layer 3 routing
could also be done only on the spine switches, or there may even be distribution
level switches that sit in between the top-of-rack switches and the routed core.
The distinguishing feature that we are trying to enable is segregating local
systems within a layer 2 domain, with routing between domains.

In a shared layer-2 architecture, the spine switches typically have to act in an
active/passive mode to act as the L3 gateway for the single shared VLAN. All
leaf switches must be attached to the active switch, and the limit on North-South
bandwidth is the connection to the active switch, so there is an upper bound on
the scalability. The Clos topology is favored because it provides horizontal
scalability. Additional spine switches can be added to increase East-West and
North-South bandwidth. Equal-cost multipath routing between switches ensures
that all links are utlized simultaneously. If all ports are full on the spine
switches, an additional tier can be added to connect additional spines,
each with their own set of leaf switches, providing hyperscale expandability.

Each network device may be taken out of service for maintenance without the entire
network being offline. This topology also allows the switches to be configured
without physical loops or Spanning Tree, since the redundant links are either
delivered via bonding or via multiple layer 3 uplink paths with equal metrics.
Some advantages of using this architecture with separate subnets per rack are:

* Reduced domain for broadcast, unknown unicast, and multicast (BUM) traffic.
* Reduced failure domain.
* Geographical separation.
* Association between IP address and rack location.
* Better cross-vendor support for multipath forwarding using equal-cost
  multipath forwarding (ECMP) via L3 routing, instead of proprietary "fabric".

This topology is significantly different from the shared-everything approach that
TripleO takes today.

====================
Problem Descriptions
====================

As this is a complex topic, it will be easier to break the problems down into
their constituent parts, based on which part of TripleO they affect:

**Problem #1: TripleO uses DHCP/PXE on the Undercloud provisioning net (ctlplane).**

Neutron on the undercloud does not yet support DHCP relays and multiple L2
subnets, since it does DHCP/PXE directly on the provisioning network.

Possible Solutions, Ideas, or Approaches:

1. Modify Ironic and/or Neutron to support multiple DHCP ranges in the dnsmasq
   configuration, use DHCP relay running on top-of-rack switches which
   receives DHCP requests and forwards them to dnsmasq on the Undercloud.
   There is a patch in progress to support that [11]_.
2. Modify Neutron to support DHCP relay. There is a patch in progress to
   support that [10]_.

Currently, if one adds a subnet to a network, Neutron DHCP agent will pick up
the changes and configure separate subnets correctly in ``dnsmasq``. For instance,
after adding a second subnet to the ``ctlplane`` network, here is the resulting
startup command for Neutron's instance of dnsmasq::

  dnsmasq --no-hosts --no-resolv --strict-order --except-interface=lo \
  --pid-file=/var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/pid \
  --dhcp-hostsfile=/var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/host \
  --addn-hosts=/var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/addn_hosts \
  --dhcp-optsfile=/var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/opts \
  --dhcp-leasefile=/var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/leases \
  --dhcp-match=set:ipxe,175 --bind-interfaces --interface=tap4ccef953-e0 \
  --dhcp-range=set:tag0,172.19.0.0,static,86400s \
  --dhcp-range=set:tag1,172.20.0.0,static,86400s \
  --dhcp-option-force=option:mtu,1500 --dhcp-lease-max=512 \
  --conf-file=/etc/dnsmasq-ironic.conf --domain=openstacklocal

The router information gets put into the dhcp-optsfile, here are the contents
of /var/lib/neutron/dhcp/aae53442-204e-4c8e-8a84-55baaeb496cf/opts::

  tag:tag0,option:classless-static-route,172.20.0.0/24,0.0.0.0,0.0.0.0/0,172.19.0.254
  tag:tag0,249,172.20.0.0/24,0.0.0.0,0.0.0.0/0,172.19.0.254
  tag:tag0,option:router,172.19.0.254
  tag:tag1,option:classless-static-route,169.254.169.254/32,172.20.0.1,172.19.0.0/24,0.0.0.0,0.0.0.0/0,172.20.0.254
  tag:tag1,249,169.254.169.254/32,172.20.0.1,172.19.0.0/24,0.0.0.0,0.0.0.0/0,172.20.0.254
  tag:tag1,option:router,172.20.0.254

The above options file will result in separate routers being handed out to
separate IP subnets. Furthermore, Neutron appears to "do the right thing" with
regard to routes for other subnets on the same network. We can see that the
option "classless-static-route" is given, with pointers to both the default
route and the other subnet(s) on the same Neutron network.

In order to modify Ironic-Inspector to use multiple subnets, we will need to
extend instack-undercloud to support network segments. There is a patch in
review to support segments in instack undercloud [0]_.

**Potential Workaround**

One possibility is to use an alternate method to DHCP/PXE boot, such as using
DHCP configuration directly on the router, or to configure a host on the remote
network which provides DHCP and PXE URLs, then provides routes back to the
ironic-conductor and metadata server as part of the DHCP response.

It is not always feasible for groups doing testing or development to configure
DHCP relay on the switches. For proof-of-concept implementations of
spine-and-leaf, we may want to configure all provisioning networks to be
trunked back to the Undercloud. This would allow the Undercloud to provide DHCP
for all networks without special switch configuration. In this case, the
Undercloud would act as a router between subnets/VLANs. This should be
considered a small-scale solution, as this is not as scalable as DHCP relay.
The configuration file for dnsmasq is the same whether all subnets are local or
remote, but dnsmasq may have to listen on multiple interfaces (today it only
listens on br-ctlplane). The dnsmasq process currently runs with
``--bind-interface=tap-XXX``, but the process will need to be run with either
binding to multiple interfaces, or with ``--except-interface=lo`` and multiple
interfaces bound to the namespace.

For proof-of-concept deployments, as well as testing environments, it might
make sense to run a DHCP relay on the Undercloud, and trunk all provisioning
VLANs back to the Undercloud. This would allow dnsmasq to listen on the tap
interface, and DHCP requests would be forwarded to the tap interface. The
downside of this approach is that the Undercloud would need to have IP
addresses on each of the trunked interfaces.

Another option is to configure dedicated hosts or VMs to be used as DHCP relay
and router for subnets on multiple VLANs, all of which would be trunked to the
relay/router host, thus acting exactly like routing switches.

------------

**Problem #2: Neutron's model for a segmented network that spans multiple L2
domains uses the segment object to allow multiple subnets to be assigned to
the same network. This functionality needs to be integrated into the
Undercloud.**

Possible Solutions, Ideas, or Approaches:

1. Implement Neutron segments on the undercloud.

The spec for Neutron routed network segments [1]_ provides a schema that we can
use to model a routed network. By implementing support for network segments, we
can provide assign Ironic nodes to networks on routed subnets. This allows us
to continue to use Neutron for IP address management, as ports are assigned by
Neutron and tracked in the Neutron database on the Undercloud. See approach #1
below.

2. Multiple Neutron networks (1 set per rack), to model all L2 segments.

By using a different set of networks in each rack, this provides us with
the flexibility to use different network architectures on a per-rack basis.
Each rack could have its own set of networks, and we would no longer have
to provide all networks in all racks. Additionally, a split-datacenter
architecture would naturally have a different set of networks in each
site, so this approach makes sense. This is detailed in approach #2 below.

3. Multiple subnets per Neutron network.

This is probably the best approach for provisioning, since Neutron is
already able to handle DHCP relay with multiple subnets as part of the
same network. Additionally, this allows a clean separation between local
subnets associated with provisioning, and networks which are used
in the overcloud, such as External networks in two different datacenters).
This is covered in more detail in approach #3 below.

4. Use another system for IPAM, instead of Neutron.

Although we could use a database, flat file, or some other method to keep
track of IP addresses, Neutron as an IPAM back-end provides many integration
benefits. Neutron integrates DHCP, hardware switch port configuration (through
the use of plugins), integration in Ironic, and other features such as
IPv6 support. This has been deemed to be infeasible due to the level of effort
required in replacing both Neutron and the Neutron DHCP server (dnsmasq).

**Approaches to Problem #2:**

Approach 1 (Implement Neutron segments on the Undercloud):

The Neutron segments model provides a schema in Neutron that allows us to
model the routed network. Using multiple subnets provides the flexibility
we need without creating exponentially more resources. We would create the same
provisioning network that we do today, but use multiple segments associated
to different routed subnets. The disadvantage to this approach is that it makes
it impossible to represent network VLANs with more than one IP subnet (Neutron
technically supports more than one subnet per port). Currently TripleO only
supports a single subnet per isolated network, so this should not be an issue.

Approach 2 (Multiple Neutron networks (1 set per rack), to model all L2 segments):

We will be using multiple networks to represent isolated networks in multiple
L2 domains. One sticking point is that although Neutron will configure multiple
routes for multiple subnets within a given network, we need to be able to both
configure static IPs and routes, and be able to scale the network by adding
additional subnets after initial deployment.

Since we control addresses and routes on the host nodes using a
combination of Heat templates and os-net-config, it is possible to use
static routes to supernets to provide L2 adjacency. This approach only
works for non-provisioning networks, since we rely on Neutron DHCP servers
providing routes to adjacent subnets for the provisioning network.

Example:
Suppose 2 subnets are provided for the Internal API network: ``172.19.1.0/24``
and ``172.19.2.0/24``. We want all Internal API traffic to traverse the Internal
API VLANs on both the controller and a remote compute node. The Internal API
network uses different VLANs for the two nodes, so we need the routes on the
hosts to point toward the Internal API gateway instead of the default gateway.
This can be provided by a supernet route to 172.19.x.x pointing to the local
gateway on each subnet (e.g. 172.19.1.1 and 172.19.2.1 on the respective
subnets). This could be represented in os-net-config with the following::

    -
      type: interface
      name: nic3
      addresses:
        -
          ip_netmask: {get_param: InternalApiIpSubnet}
      routes:
        -
          ip_netmask: {get_param: InternalApiSupernet}
          next_hop: {get_param: InternalApiRouter}

Where InternalApiIpSubnet is the IP address on the local subnet,
InternalApiSupernet is '172.19.0.0/16', and InternalApiRouter is either
172.19.1.1 or 172.19.2.1 depending on which local subnet the host belongs to.

The end result of this is that each host has a set of IP addresses and routes
that isolate traffic by function. In order for the return traffic to also be
isolated by function, similar routes must exist on both hosts, pointing to the
local gateway on the local subnet for the larger supernet that contains all
Internal API subnets.

The downside of this is that we must require proper supernetting, and this may
lead to larger blocks of IP addresses being used to provide ample space for
scaling growth. For instance, in the example above an entire /16 network is set
aside for up to 255 local subnets for the Internal API network. This could be
changed into a more reasonable space, such as /18, if the number of local
subnets will not exceed 64, etc. This will be less of an issue with native IPv6
than with IPv4, where scarcity is much more likely.

Approach 3 (Multiple subnets per Neutron network):

The approach we will use for the provisioning network will be to use multiple
subnets per network, using Neutron segments. This will allow us to take
advantage of Neutron's ability to support multiple networks with DHCP relay.
The DHCP server will supply the necessary routes via DHCP until the nodes are
configured with a static IP post-deployment.

---------

**Problem #3: Ironic introspection DHCP doesn't yet support DHCP relay**

This makes it difficult to do introspection when the hosts are not on the same L2
domain as the controllers. Patches are either merged or in review to support
DHCP relay.

Possible Solutions, Ideas, or Approaches:

1. A patch to support a dnsmasq PXE filter driver has been merged. This will
   allow us to support selective DHCP when using DHCP relay (where the packet
   is not coming from the MAC of the host but rather the MAC of the switch)
   [12]_.

2. A patch has been merged to puppet-ironic to support multiple DHCP subnets
   for Ironic Inspector [13]_.

3. A patch is in review to add support for multiple subnets for the
   provisioning network in the instack-undercloud scripts [14]_.

For more information about solutions, please refer to the
tripleo-routed-networks-ironic-inspector blueprint [5]_ and spec [6]_.

-------

**Problem #4: The IP addresses on the provisioning network need to be
static IPs for production.**

Possible Solutions, Ideas, or Approaches:

1. Dan Prince wrote a patch [9]_ in Newton to convert the ctlplane network
   addresses to static addresses post-deployment. This will need to be
   refactored to support multiple provisioning subnets across routers.

Solution Implementation

This work is done and merged for the legacy use case. During the
initial deployment, the nodes receive their IP address via DHCP, but during
Heat deployment the os-net-config script is called, which writes static
configuration files for the NICs with static IPs.

This work will need to be refactored to support assigning IPs from the
appropriate subnet, but the work will be part of the TripleO Heat Template
refactoring listed in Problems #6, and #7 below.

For the deployment model where the IPs are specified (ips-from-pool-all.yaml),
we need to develop a model where the Control Plane IP can be specified
on multiple deployment subnets. This may happen in a later cycle than the
initial work being done to enable routed networks in TripleO. For more
information, reference the tripleo-predictable-ctlplane-ips blueprint [7]_
and spec [8]_.

------

**Problem #5: Heat Support For Routed Networks**

The Neutron routed networks extensions were only added in recent releases, and
there was a dependency on TripleO Heat Templates.

Possible Solutions, Ideas or Approaches:

1. Add the required objects to Heat. At minimum, we will probably have to
   add ``OS::Neutron::Segment``, which represents layer 2 segments, the
   ``OS::Neutron::Network`` will be updated to support the ``l2-adjacency``
   attribute, ``OS::Neutron::Subnet`` and ``OS::Neutron:port`` would be extended
   to support the ``segment_id`` attribute.

Solution Implementation:

Heat now supports the OS::Neutron::Segment resource. For example::

  heat_template_version: 2015-04-30
  ...
  resources:
    ...
    the_resource:
      type: OS::Neutron::Segment
      properties:
        description: String
        name: String
        network: String
        network_type: String
        physical_network: String
        segmentation_id: Integer

This work has been completed in Heat with this review [15]_.

------

**Problem #6: Static IP assignment: Choosing static IPs from the correct
subnet**

Some roles, such as Compute, can likely be placed in any subnet, but we will
need to keep certain roles co-located within the same set of L2 domains. For
instance, whatever role is providing Neutron services will need all controllers
in the same L2 domain for VRRP to work properly.

The network interfaces will be configured using templates that create
configuration files for os-net-config. The IP addresses that are written to each
node's configuration will need to be on the correct subnet for each host. In
order for Heat to assign ports from the correct subnets, we will need to have a
host-to-subnets mapping.

Possible Solutions, Ideas or Approaches:

1. The simplest implementation of this would probably be a mapping of role/index
   to a set of subnets, so that it is known to Heat that Controller-1 is in
   subnet set X and Compute-3 is in subnet set Y.
2. We could associate particular subnets with roles, and then use one role
   per L2 domain (such as per-rack).
3. The roles and templates should be refactored to allow for dynamic IP
   assignment within subnets associated with the role. We may wish to evaluate
   the possibility of storing the routed subnets in Neutron using the routed
   networks extensions that are still under development. This would provide
   additional flexibility, but is probably not required to implement separate
   subnets in each rack.
4. A scalable long-term solution is to map which subnet the host is on
   during introspection. If we can identify the correct subnet for each
   interface, then we can correlate that with IP addresses from the correct
   allocation pool.  This would have the advantage of not requiring a static
   mapping of role to node to subnet. In order to do this, additional
   integration would be required between Ironic and Neutron (to make Ironic
   aware of multiple subnets per network, and to add the ability to make
   that association during introspection).

Solution Impelementation:

Solutions 1 and 2 above have been implemented in the "composable roles" series
of patches [16]_. The initial implementation uses separate Neutron networks
for different L2 domains. These templates are responsible for assigning the
isolated VLANs used for data plane and overcloud control planes, but does not
address the provisioning network. Future work may refactor the non-provisioning
networks to use segments, but for now non-provisioning networks must use
different networks for different roles.

Ironic autodiscovery may allow us to determine the subnet where each node
is located without manual entry. More work is required to automate this
process.

------

**Problem #7: Isolated Networking Requires Static Routes to Ensure Correct VLAN
is Used**

In order to continue using the Isolated Networks model, routes will need to be
in place on each node, to steer traffic to the correct VLAN interfaces. The
routes are written when os-net-config first runs, but may change. We
can't just rely on the specific routes to other subnets, since the number of
subnets will increase or decrease as racks are added or taken away. Rather than
try to deal with constantly changing routes, we should use static routes that
will not need to change, to avoid disruption on a running system.

Possible Solutions, Ideas or Approaches:

1. Require that supernets are used for various network groups. For instance,
   all the Internal API subnets would be part of a supernet, for instance
   172.17.0.0/16 could be used, and broken up into many smaller subnets, such
   as /24. This would simplify the routes, since only a single route for
   172.17.0.0/16 would be required pointing to the local router on the
   172.17.x.0/24 network.
2. Modify os-net-config so that routes can be updated without bouncing
   interfaces, and then run os-net-config on all nodes when scaling occurs.
   A review for this functionality was considered and abandeded [3]_.
   The patch was determined to have the potential to lead to instability.

os-net-config configures static routes for each interface. If we can keep the
routing simple (one route per functional network), then we would be able to
isolate traffic onto functional VLANs like we do today.

It would be a change to the existing workflow to have os-net-config run on
updates as well as deployment, but if this were a non-impacting event (the
interfaces didn't have to be bounced), that would probably be OK.

At a later time, the possibility of using dynamic routing should be considered,
since it reduces the possibility of user error and is better suited to
centralized management. SDN solutions are one way to provide this, or other
approaches may be considered, such as setting up OVS tunnels.

Proposed Change
===============
The proposed changes are discussed below.

Overview
--------

In order to provide spine-and-leaf networking for deployments, several changes
will have to be made to TripleO:

1. Support for DHCP relay in Ironic and Neutron DHCP servers. Implemented in
   patch [15]_ and the patch series [17]_.
2. Refactoring of TripleO Heat Templates network isolation to support multiple
   subnets per isolated network, as well as per-subnet and supernet routes.
   The bulk of this work is done in the patch series [16]_ and in patch [10]_.
3. Changes to Infra CI to support testing.
4. Documentation updates.

Alternatives
------------

The approach outlined here is very prescriptive, in that the networks must be
known ahead of time, and the IP addresses must be selected from the appropriate
pool. This is due to the reliance on static IP addresses provided by Heat.

One alternative approach is to use DHCP servers to assign IP addresses on all
hosts on all interfaces. This would simplify configuration within the Heat
templates and environment files. Unfortunately, this was the original approach
of TripleO, and it was deemed insufficient by end-users, who wanted stability
of IP addresses, and didn't want to have an external dependency on DHCP.

Another approach is to use the DHCP server functionality in the network switch
infrastructure in order to PXE boot systems, then assign static IP addresses
after the PXE boot is done via DHCP. This approach only solves for part of the
requirement: the net booting. It does not solve the desire to have static IP
addresses on each network. This could be achieved by having static IP addresses
in some sort of per-node map. However, this approach is not as scalable as
programatically determining the IPs, since it only applies to a fixed number of
hosts. We want to retain the ability of using Neutron as an IP address
management (IPAM) back-end, ideally.

Another approach which was considered was simply trunking all networks back
to the Undercloud, so that dnsmasq could respond to DHCP requests directly,
rather than requiring a DHCP relay. Unfortunately, this has already been
identified as being unacceptable by some large operators, who have network
architectures that make heavy use of L2 segregation via routers. This also
won't work well in situations where there is geographical separation between
the VLANs, such as in split-site deployments.

Security Impact
---------------

One of the major differences between spine-and-leaf and standard isolated
networking is that the various subnets are connected by routers, rather than
being completely isolated. This means that without proper ACLs on the routers,
networks which should be private may be opened up to outside traffic.

This should be addressed in the documentation, and it should be stressed that
ACLs should be in place to prevent unwanted network traffic. For instance, the
Internal API network is sensitive in that the database and message queue
services run on that network. It is supposed to be isolated from outside
connections. This can be achieved fairly easily if *supernets* are used, so
that if all Internal API subnets are a part of the ``172.19.0.0/16`` supernet,
an ACL rule will allow only traffic between Internal API IPs (this is a
simplified example that could be applied to any Internal API VLAN, or as a
global ACL)::

  allow traffic from 172.19.0.0/16 to 172.19.0.0/16
  deny traffic from * to 172.19.0.0/16

Other End User Impact
---------------------

Deploying with spine-and-leaf will require additional parameters to
provide the routing information and multiple subnets required. This will have
to be documented. Furthermore, the validation scripts may need to be updated
to ensure that the configuration is validated, and that there is proper
connectivity between overcloud hosts.

Performance Impact
------------------

Much of the traffic that is today made over layer 2 will be traversing layer
3 routing borders in this design. That adds some minimal latency and overhead,
although in practice the difference may not be noticeable. One important
consideration is that the routers must not be too overcommitted on their
uplinks, and the routers must be monitored to ensure that they are not acting
as a bottleneck, especially if complex access control lists are used.

Other Deployer Impact
---------------------

A spine-and-leaf deployment will be more difficult to troubleshoot than a
deployment that simply uses a set of VLANs. The deployer may need to have
more network expertise, or a dedicated network engineer may be needed to
troubleshoot in some cases.

Developer Impact
----------------

Spine-and-leaf is not easily tested in virt environments. This should be
possible, but due to the complexity of setting up libvirt bridges and
routes, we may want to provide a simulation of spine-and-leaf for use in
virtual environments. This may involve building multiple libvirt bridges
and routing between them on the Undercloud, or it may involve using a
DHCP relay on the virt-host as well as routing on the virt-host to simulate
a full routing switch. A plan for development and testing will need to be
developed, since not every developer can be expected to have a routed
environment to work in. It may take some time to develop a routed virtual
environment, so initial work will be done on bare metal.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Dan Sneddon <dsneddon@redhat.com>

Approver(s)
-----------

Primary approver:
  Emilien Macchi <emacchi@redhat.com>

Work Items
----------

1. Add static IP assignment to Control Plane [DONE]
2. Modify Ironic Inspector ``dnsmasq.conf`` generation to allow export of
   multiple DHCP ranges, as described in Problem #1 and Problem #3.
3. Evaluate the Routed Networks work in Neutron, to determine if it is required
   for spine-and-leaf, as described in Problem #2.
4. Add OS::Neutron::Segment and l2-adjacency support to Heat, as described
   in Problem #5. This may or may not be a dependency for spine-and-leaf, based
   on the results of work item #3.
5. Modify the Ironic-Inspector service to record the host-to-subnet mappings,
   perhaps during introspection, to address Problem #6.
6. Add parameters to Isolated Networking model in Heat to support supernet
   routes for individual subnets, as described in Problem #7.
7. Modify Isolated Networking model in Heat to support multiple subnets, as
   described in Problem #8.
8. Add support for setting routes to supernets in os-net-config NIC templates,
   as described in the proposed solution to Problem #2.
9. Implement support for iptables on the Controller, in order to mitigate
   the APIs potentially being reachable via remote routes. Alternatively,
   document the mitigation procedure using ACLs on the routers.
10. Document the testing procedures.
11. Modify the documentation in tripleo-docs to cover the spine-and-leaf case.


Implementation Details
----------------------

Workflow:

1. Operator configures DHCP networks and IP address ranges
2. Operator imports baremetal instackenv.json
3. When introspection or deployment is run, the DHCP server receives the DHCP
   request from the baremetal host via DHCP relay
4. If the node has not been introspected, reply with an IP address from the
   introspection pool* and the inspector PXE boot image
5. If the node already has been introspected, then the server assumes this is
   a deployment attempt, and replies with the Neutron port IP address and the
   overcloud-full deployment image
6. The Heat templates are processed which generate os-net-config templates, and
   os-net-config is run to assign static IPs from the correct subnets, as well
   as routes to other subnets via the router gateway addresses.

* The introspection pool will be different for each provisioning subnet.

When using spine-and-leaf, the DHCP server will need to provide an introspection
IP address on the appropriate subnet, depending on the information contained in
the DHCP relay packet that is forwarded by the segment router. dnsmasq will
automatically match the gateway address (GIADDR) of the router that forwarded
the request to the subnet where the DHCP request was received, and will respond
with an IP and gateway appropriate for that subnet.

The above workflow for the DHCP server should allow for provisioning IPs on
multiple subnets.

Dependencies
============

There may be a dependency on the Neutron Routed Networks. This won't be clear
until a full evaluation is done on whether we can represent spine-and-leaf
using only multiple subnets per network.

There will be a dependency on routing switches that perform DHCP relay service
for production spine-and-leaf deployments.

Testing
=======

In order to properly test this framework, we will need to establish at least
one CI test that deploys spine-and-leaf. As discussed in this spec, it isn't
necessary to have a full routed bare metal environment in order to test this
functionality, although there is some work to get it working in virtual
environments such as OVB.

For bare metal testing, it is sufficient to trunk all VLANs back to the
Undercloud, then run DHCP proxy on the Undercloud to receive all the
requests and forward them to br-ctlplane, where dnsmasq listens. This
will provide a substitute for routers running DHCP relay. For Neutron
DHCP, some modifications to the iptables rule may be required to ensure
that all DHCP requests from the overcloud nodes are received by the
DHCP proxy and/or the Neutron dnsmasq process running in the dhcp-agent
namespace.

Documentation Impact
====================

The procedure for setting up a dev environment will need to be documented,
and a work item mentions this requirement.

The TripleO docs will need to be updated to include detailed instructions
for deploying in a spine-and-leaf environment, including the environment
setup. Covering specific vendor implementations of switch configurations
is outside this scope, but a specific overview of required configuration
options should be included, such as enabling DHCP relay (or "helper-address"
as it is also known) and setting the Undercloud as a server to receive
DHCP requests.

The updates to TripleO docs will also have to include a detailed discussion
of choices to be made about IP addressing before a deployment. If supernets
are to be used for network isolation, then a good plan for IP addressing will
be required to ensure scalability in the future.

References
==========

.. [0] `Review: TripleO Heat Templates: Tripleo routed networks ironic inspector, and Undercloud <https://review.openstack.org/#/c/437544>`_
.. [1] `Spec: Routed Networks for Neutron <https://specs.openstack.org/openstack/neutron-specs/specs/newton/routed-networks.html>`_
.. [3] `Review: Modify os-net-config to make changes without bouncing interface <https://review.openstack.org/#/c/152732/>`_
.. [5] `Blueprint: Modify TripleO Ironic Inspector to PXE Boot Via DHCP Relay <https://blueprints.launchpad.net/tripleo/+spec/tripleo-routed-networks-ironic-inspector>`_
.. [6] `Spec: Modify TripleO Ironic Inspector to PXE Boot Via DHCP Relay <https://review.openstack.org/#/c/421011>`_
.. [7] `Blueprint: User-specifiable Control Plane IP on TripleO Routed Isolated Networks <https://blueprints.launchpad.net/tripleo/+spec/tripleo-routed-networks-deployment>`_
.. [8] `Spec: User-specifiable Control Plane IP on TripleO Routed Isolated Networks <https://review.openstack.org/#/c/421010>`_
.. [9] `Review: Configure ctlplane network with a static IP <https://review.openstack.org/#/c/206022/>`_
.. [10] `Review: Neutron: Make "on-link" routes for subnets optional <https://review.openstack.org/#/c/438171>`_
.. [11] `Review: Ironic Inspector: Make "on-link" routes for subnets optional <https://review.openstack.org/438175>`_
.. [12] `Review: Ironic Inspector: Introducing a dnsmasq PXE filter driver <https://review.openstack.org/466448>`_
.. [13] `Review: Multiple DHCP Subnets for Ironic Inspector <https://review.openstack.org/#/c/436716>`_
.. [14] `Review: Instack Undercloud: Add support for multiple inspection subnets <https://review.openstack.org/#/c/533367>`_
.. [15] `Review: DHCP Agent: Separate local from non-local subnets <https://review.openstack.org/#/c/468744>`_
.. [16] `Review Series: topic:bp/composable-networks <https://review.openstack.org/#/q/topic:bp/composable-networks+(status:open+OR+status:merged)>`_
.. [17] `Review Series: project:openstack/networking-baremetal <https://review.openstack.org/#/q/project:openstack/networking-baremetal+committer:hjensas%2540redhat.com>`_

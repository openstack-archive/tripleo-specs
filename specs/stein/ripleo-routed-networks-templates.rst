..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================================
TripleO Routed Networks Deployment (Spine-and-Leaf Clos)
========================================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-routed-networks-templates

This blueprint is part of a the series tripleo-routed-networks-deployment [0]_.

TripleO uses shared L2 networks for all networks except the provisioning
network today. (Support for L3 provisioning network where added in Queens.)

L3 support on the provisioning network is using network segments, a concept
in Neutron routed networks, we can represent more than one subnet per VLAN.
Without network segments, we would be limited to one subnet per VLAN.

For the non-provisioning networks we have no way to model a true L3 routed
network in TripleO today. When deploying such an architecture we currently
create custom (neutron) networks for all the different l2 segments for each
isolated network. While this approach works it comes with some caveats.

This spec covers refactoring the TripleO Heat Templates to support deployment
onto networks which are segregated into multiple layer 2 domains with routers
forwarding traffic between layer 2 domains.


Problem Description
===================

The master blueprint for routed networks for deployments breaks the problem
set into multiple parts [0]_. This blueprint presents the problems which are
applicable to this blueprint below.


====================
Problem Descriptions
====================


Problem #1: Deploy systems onto a routed provisioning network.

While we can model a routed provisioning network and deploy systems on top of
that network today. Doing so requires additional complex configuration, such
as:

 * Setting up the required static routes to ensure traffic within the L3
   control plane takes the desired path troughout the network.
 * L2 segments use different router addresses.
 * L2 segments may use different subnet masks.
 * Other L2 segment property differences.


This configuration is essentially manually passing in information in the
templates to deploy the overcloud. Information that was already provided when
deploying the undercloud. While this works, it increases complexity and the
possibility that the user provides incorrect configuration data.

We should be able to get as much of this information based on what was provided
when deploying the undercloud.

In order to support this model, there are some requirements that have to be
met in Heat and Neutron.

**Alternative approaches to Problem #1:**


Approach 1:

.. NOTE:: This is what we currently do.

Since we control addresses and routes on the host nodes using a
combination of Heat templates and os-net-config, it may be possible to use
static routes to supernets to provide L2 adjacency, rather than relying on
Neutron to generate dynamic lists of routes that would need to be updated
on all hosts.

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

Approch 2:

Instead of passing parameters such as ControlPlaneCidr,
ControlPlaneDefaultRoute etc implement Neutron RFE [5]_ and Heat RFE [6]_. In
tripleo-heat-templates we can then use get_attr to get the data. And we leave
it to neutron to calculate and provide the routes for the L3 network.

This would require [3]_, which I believe was in quite good shape before it was
abandoned due to activity policy. (An alternative would be to change
os-net-config to have an option to only change and apply routing configuration.
Something like running `ifdown-routes
<https://github.com/fedora-sysv/initscripts/blob/master/sysconfig/network-scripts/ifdown-routes>`_
/
`ifup-routes
<https://github.com/fedora-sysv/initscripts/blob/master/sysconfig/network-scripts/ifup-routes>`_
, however [3]_ is likely the better solution.)


------

**Problem #2: Static IP assignment: Choosing static IPs from the correct
subnet**

Some roles, such as Compute, can likely be placed in any subnet, but we will
need to keep certain roles co-located within the same set of L2 domains. For
instance, whatever role is providing Neutron services will need all controllers
in the same L2 domain for VRRP to work properly.

The network interfaces will be configured using templates that create
configuration files for os-net-config. The IP addresses that are written to
each node's configuration will need to be on the correct subnet for each host.
In order for Heat to assign ports from the correct subnets, we will need to
have a host-to-subnets mapping.

Possible Solutions, Ideas or Approaches:

.. NOTE:: We currently use #2, by specifying parameters for each role.

1. The simplest implementation of this would probably be a mapping of
   role/index to a set of subnets, so that it is known to Heat that
   Controller-1 is in subnet set X and Compute-3 is in subnet set Y. The node
   would then have the ip and subnet info for each network chosen from the
   appropriate set of subnets. For other nodes, we would need to
   programatically determine which subnets are correct for a given node.
2. We could associate particular subnets with roles, and then use one role
   per L2 domain (such as per-rack). This might be achieved with a map of
   roles to subnets, or by specifying parameters for each role such as:
   supernet, subnet (ID and/or ip/netmask), and subnet router.
3. Initial implementation might follow the model for isolated networking
   demonstrated by the environments/ips-from-pool-all.yaml. Developing the
   ips-from-pool model first will allow testing various components with
   spine-and-leaf while the templates that use dynamic assignment of IPs
   within specified subnets are developed.
4. The roles and templates should be refactored to allow for dynamic IP
   assignment within subnets associated with the role. We may wish to evaluate
   the possibility of storing the routed subnets in Neutron using the routed
   networks extensions that are still under development. However, in this
   case, This is probably not required to implement separate subnets in each
   rack.
5. A scalable long-term solution is to map which subnet the host is on
   during introspection. If we can identify the correct subnet for each
   interface, then we can correlate that with IP addresses from the correct
   allocation pool.  This would have the advantage of not requiring a static
   mapping of role to node to subnet. In order to do this, additional
   integration would be required between Ironic and Neutron (to make Ironic
   aware of multiple subnets per network, and to add the ability to make
   that association during introspection.

We will also need to take into account sitations where there are heterogeneous
hardware nodes in the same layer 2 broadcast domain (such as within a rack).

.. Note:: This can be done either using node groups in NetConfigDataLookup as
          implemented in review [4]_ or by using additional custom roles.

------

**Problem #3: Isolated Networking Requires Static Routes to Ensure Correct VLAN
is Used**

In order to continue using the Isolated Networks model, routes will need to be
in place on each node, to steer traffic to the correct VLAN interfaces. The
routes are written when os-net-config first runs, but may change. We
can't just rely on the specific routes to other subnets, since the number of
subnets will increase or decrease as racks are added or taken away.

Possible Solutions, Ideas or Approaches:

1. Require that supernets are used for various network groups. For instance,
   all the Internal API subnets would be part of a supernet, for instance
   172.17.0.0/16 could be used, and broken up into many smaller subnets, such
   as /24. This would simplify the routes, since only a single route for
   172.17.0.0/16 would be required pointing to the local router on the
   172.17.x.0/24 network.

   Example:
   Suppose 2 subnets are provided for the Internal API network: 172.19.1.0/24
   and 172.19.2.0/24. We want all Internal API traffic to traverse the Internal
   API VLANs on both the controller and a remote compute node. The Internal API
   network uses different VLANs for the two nodes, so we need the routes on the
   hosts to point toward the Internal API gateway instead of the default
   gateway. This can be provided by a supernet route to 172.19.x.x pointing to
   the local gateway on each subnet (e.g. 172.19.1.1 and 172.19.2.1 on the
   respective subnets). This could be represented in an os-net-config with the
   following::

    -
      type: interface
      name: nic3
      addresses:
        -
          ip_netmask: {get_param: InternalApiXIpSubnet}
      routes:
        -
          ip_netmask: {get_param: InternalApiSupernet}
          next_hop: {get_param: InternalApiXDefaultRoute}

   Where InternalApiIpSubnet is the IP address on the local subnet,
   InternalApiSupernet is '172.19.0.0/16', and InternalApiRouter is either
   172.19.1.1 or 172.19.2.1 depending on which local subnet the host belongs to.
2. Modify os-net-config so that routes can be updated without bouncing
   interfaces, and then run os-net-config on all nodes when scaling occurs.
   A review for this functionality is in progress [3]_.
3. Instead of passing parameters to THT about routes (or supernet routes),
   implement Neutron RFE [5]_ and Heat RFE [6]_. In tripleo-heat-templates we
   can then use get_attr to get the data we currently read from user provided
   parameters such as the InternalApiSupernet and InternalApiXDefaultRoute in
   the example above. (We might also consider replacing [6]_ with a change
   extending the ``network/ports/port.j2`` in tripleo-heat-templates to output
   this data.)

os-net-config configures static routes for each interface. If we can keep the
routing simple (one route per functional network), then we would be able to
isolate traffic onto functional VLANs like we do today.

It would be a change to the existing workflow to have os-net-config run on
updates as well as deployment, but if this were a non-impacting event (the
interfaces didn't have to be bounced), that would probably be OK. (An
alternative is to add an option to have an option in os-net-config that only
adds new routes. Something like, os-net-config --no-activate +
ifdown-routes/ifup-routes.)

At a later time, the possibility of using dynamic routing should be considered,
since it reduces the possibility of user error and is better suited to
centralized management. The overcloud nodes might participate in internal
routing protocols. SDN solutions are another way to provide this, or other
approaches may be considered, such as setting up OVS tunnels.

------

**Problem #4: Isolated Networking in TripleO Heat Templates Needs to be
Refactored**

The current isolated networking templates use parameters in nested stacks to
define the IP information for each network. There is no room in the current
schema to define multiple subnets per network, and no way to configure the
routers for each network. These values are provided by single parameters.

Possible Solutions, Ideas or Approaches:

1. We would need to refactor these resources to provide different routers
   for each network.
2. We extend the custom and isolated networks in TripleO to add support for
   Neutron routed-networks (segments) and multiple subnets. Each subnet will be
   mapped to a different L2 segment. We should make the extension backward
   compatible and only enable Neutron routed-networks (I.e associate subnets
   with segments.) when the templates used define multiple subnets on a
   network. To enable this we need some changes to land in Neutron and Heat,
   these are the in-progress reviews:

     * Allow setting network-segment on subnet update [7]_
     * Allow updating the segment property of OS::Neutron::Subnet [8]_
     * Add first_segment convenience attr to OS::Neutron::Net [9]_



Proposed Change
===============
The proposed changes are discussed below.

Overview
--------

In order to provide spine-and-leaf networking for deployments, several changes
will have to be made to TripleO:

1. Support for DHCP relay in Neutron DHCP servers (in progress), and Ironic
   DHCP servers (this is addressed in separate blueprints in the same series).
2. Refactor assignment of Control Plane IPs to support routed networks (that
   is addressed by a separate blueprint: tripleo-predictable-ctlplane-ips [2]_.
3. Refactoring of TripleO Heat Templates network isolation to support multiple
   subnets per isolated network, as well as per-subnet and supernet routes.
4. Changes to Infra CI to support testing.
5. Documentation updates.

Alternatives
------------

The approach outlined here is very prescriptive, in that the networks must be
known ahead of time, and the IP addresses must be selected from the appropriate
pool. This is due to the reliance on static IP addresses provided by Heat.
Heat will have to model the subnets and associate them with roles (node
groups).

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
connections. This can be achieved fairly easily if supernets are used, so that
if all Internal API subnets are a part of the 172.19.0.0/16 supernet, a simple
ACL rule will allow only traffic between Internal API IPs (this is a simplified
example that would be generally applicable to all Internal API router VLAN
interfaces or for a global ACL)::

  allow traffic from 172.19.0.0/16 to 172.19.0.0/16
  deny traffic from * to 172.19.0.0/16

The isolated networks design separates control plane traffic from data plane
traffic, and separates administrative traffic from tenant traffic. In order
to preserve this separatation of traffic, we will use static routes pointing
to supernets. This ensures all traffic to any subnet within a network will exit
via the interface attached to the local subnet in that network. It will be
important for the end user to implement ACLs in a routed network to prevent
remote access to networks that would be completely isolated in a shared L2
deployment.

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
routes, we may want to provide a pre-configured quickstart environment
for testing. This may involve building multiple libvirt bridges
and routing between them on the Undercloud, or it may involve using a
DHCP relay on the virt-host as well as routing on the virt-host to simulate
a full routing switch. A plan for development and testing will need to be
developed, since not every developer can be expected to have a routed
environment to work in. It may take some time to develop a routed virtual
environment, so initial work will be done on bare metal.

A separate blueprint will cover adding routed network support to
tripleo-quickstart.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  - Dan Sneddon <dsneddon@redhat.com>

Other assignees:
  - Bob Fournier <bfournie@redhat.com>
  - Harald Jensas <hjensas@redhat.com>
  - Steven Hardy <shardy@redhat.com>
  - Dan Prince <dprince@redhat.com>

Approver(s)
-----------

Primary approver:
  Alex Schultz <aschultz@redhat.com>

Work Items
----------

1. Implement support for DHCP on routed networks using DHCP relay, as
   described in Problem #1 above.
2. Add parameters to Isolated Networking model in Heat to support supernet
   routes for individual subnets, as described in Problem #3.
3. Modify Isolated Networking model in Heat to support multiple subnets, as
   described in Problem #4.
4. Implement support for iptables on the Controller, in order to mitigate
   the APIs potentially being reachable via remote routes, as described in
   the Security Impact section. Alternatively, document the mitigation
   procedure using ACLs on the routers.
5. Document the testing procedures.
6. Modify the documentation in tripleo-docs to cover the spine-and-leaf case.
7. Modify the Ironic-Inspector service to record the host-to-subnet mappings,
   perhaps during introspection, to address Problem #2 (long-term).


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

When using spine-and-leaf, the DHCP server will need to provide an
introspection IP address on the appropriate subnet, depending on the
information contained in the DHCP relay packet that is forwarded by the segment
router. dnsmasq will automatically match the gateway address (GIADDR) of the
router that forwarded the request to the subnet where the DHCP request was
received, and will respond with an IP and gateway appropriate for that subnet.

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

.. [0] `Blueprint: TripleO Routed Networks for Deployments <https://blueprints.launchpad.net/tripleo/+spec/tripleo-routed-networks-deployment>`_
.. [2] `Spec: User-specifiable Control Plane IP on TripleO Routed Isolated Networks <https://review.openstack.org/#/c/421010/>`_
.. [3] `Review: Modify os-net-config to make changes without bouncing interface <https://review.openstack.org/#/c/152732/>`_
.. [4] `Review: Add support for node groups in NetConfigDataLookup <https://review.openstack.org/#/c/406641/>`_
.. [5] `[RFE] Create host-routes for routed networks (segments) <https://bugs.launchpad.net/neutron/+bug/1766380>`_
.. [6] `[RFE] Extend attributes of Server and Port resource to client interface configuration data <https://storyboard.openstack.org/#!/story/1766946>`_
.. [7] `Allow setting network-segment on subnet update <https://review.openstack.org/523972>`_
.. [8] `Allow updating the segment property of OS::Neutron::Subnet <https://review.openstack.org/567206>`_
.. [9] `Add first_segment convenience attr to OS::Neutron::Net <https://review.openstack.org/567207>`_

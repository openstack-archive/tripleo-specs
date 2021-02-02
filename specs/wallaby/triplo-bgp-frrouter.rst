..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Install and Configure FRRouter
==============================

The goal of this spec is to design and plan requirements for adding support to
TripleO to install and provide a basic configuration of Free Range Router (FRR)
on overcloud nodes in order to support BGP dynamic routing. There are multiple
reasons why an administrator might want to run FRR, including to obtain
multiple routes on multiple uplinks to northbound switches, or to advertise
routes to networks or IP addresses via dynamic routing protocols.

Problem description
===================

There are several use cases for using BGP, and in fact there are separate
efforts underway to utilize BGP for the control plane and data plane.

BGP may be used for equal-cost multipath (ECMP) load balancing of outbound
links, and bi-directional forwarding detection (BFD) for resiliency to ensure
that a path provides connectivity. For outbound connectivity BGP will learn
routes from BGP peers.

BGP may be used for advertising routes to API endpoints. In this model HAProxy
will listen on an IP address and FRR will advertise routes to that IP to BGP
peers. High availability for HAProxy is provided via other means such as
Pacemaker, and FRR will simply advertise the virtual IP address when it is
active on an API controller.

BGP may also be used for routing inbound traffic to provider network IPs or
floating IPs for instance connectivity. The Compute nodes will run FRR to
advertise routes to the local VM IPs or floating IPs hosted on the node. FRR
has a daemon named Zebra that is responsible for exchanging routes between
routing daemons such as BGP and the kernel. The *redistribute connected*
statement in the FRR configuration will cause local IP addresses on the host
to be advertised via BGP. Floating IP addresses are attached to a loopback
interface in a namespace, so they will be redistributed using this method.
Changes to OVN will be required to ensure provider network IPs assigned to VMs
will be assigned to a loopback interface in a namespace in a similar fashion.

Proposed Change
===============

Overview
--------

Create a container with FRR. The container will run the BGP daemon, BFD
daemon, and Zebra daemon (which copies routes to/from the kernel). Provide a
basic configuration that would allow BGP peering with multiple peers. In the
control plane use case the FRR container needs to be started along with the HA
components, but in the data plane use case the container will be a sidecar
container supporting Neutron. The container is defined in a change proposed
here: [1]_

The container will be deployed using a TripleO Deployment Service. The service
will use Ansible to template the FRR configuration file, and a simple
implementation exists in a proposed change here: [2]_

The current FRR Ansible module is insufficient to configure BGP parameters and
would need to be extended. At this time the Ansible Networking development
team is not interested in extending the FRR module, so the configuration will
be provided using TripleO templates for the FRR main configuration file and
daemon configuration file. Those templates are defined in a change proposed
here: [3]_

A user-modifiable environment file will need to be provided so the installer
can provide the configuration data needed for FRR (see User Experience below).

OVN will need to be modified to enable the Compute node to assign VM provider
network IPs to a loopback interface inside a namespace. These IP address will
not be used for sending or receiving traffic, only for redistributing routes
to the IPs to BGP peers. Traffic which is sent to those IP addresses will be
forwarded to the VM using OVS flows on the hypervisor.  An example agent for
OVN has been written to demonstrate how to monitor the southbound OVN DB and
create loopback IP addresses when a VM is started on a Compute node. The OVN
changes will be detailed in a separate OVN spec. Demonstration code is
available on Github: [4]_

User Experience
^^^^^^^^^^^^^^^

The installer will need to provide some basic information for the FRR
configuration, including whether to enable BFD, BGP IPv4, BGP IPv6,
and other settings. See the Example Configuration Data section below.

Additional user-provided data may include inbound or outbound filter prefixes.
The default filter prefixes will accept only default routes via BGP, and will
export only loopback IPs, which have a /32 subnet mask for IPv4 or /128 subnet
mask for IPv6.

Example Configuration Data
^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

   tripleo_frr_bfd: false
   tripleo_frr_bgp: false
   tripleo_frr_bgp_ipv4: true
   tripleo_frr_bgp_ipv4_allowas_in: false
   tripleo_frr_bgp_ipv6: true
   tripleo_frr_bgp_ipv6_allowas_in: false
   tripleo_frr_config_basedir: "/var/lib/config-data/ansible-generated/frr"
   tripleo_frr_hostname: "{{ ansible_hostname }}"
   tripleo_frr_log_level: informational
   tripleo_frr_watchfrr: true
   tripleo_frr_zebra: false

Alternatives
============

1. Routing outbound traffic via multiple uplinks

   Fault-tolerance and load-balancing for outbound traffic is typically
   provided by bonding Ethernet interfaces. This works for most cases, but
   is susceptible to unidirectional interface failure, a situation where
   traffic works in only one direction. The LACP protocol for bonding does
   provide some protection against unidirectional traffic failures, but is not
   as robust as bi-directional forwarding detection (BFD) provided by FRR.

2. Routing inbound traffic to highly-available API endpoints

   The most common method currently used to provide HA for API endpoints is
   to use a virtual IP that fails over from active to standby nodes using a
   shared Ethernet MAC address. The drawback to this method is that all
   standby API controllers must reside on the same layer 2 segment (VLAN) as
   the active controller. This presents a challenge if the operator wishes
   to place API controllers in different failure domains for power and/or
   networking. A BGP daemon avoids this limitation by advertising a route
   to the shared IP address directly to the BGP peering router over a routed
   layer 3 link.


3. Routing to Neutron IP addresses

   Data plane traffic is usually delivered to provider network or floating
   IP addresses via the Ethernet MAC address associated with the IP and
   determined via ARP requests on a shared VLAN. This requires that every
   Compute node which may host a provider network IP or floating IP has
   the appropriate VLAN trunked to a provider bridge attached to an interface
   or bond. This makes it impossible to migrate VMs or floating IPs across
   layer 3 boundaries in edge computing topologies or in a fully layer 3
   routed datacenter.


Security Impact
===============

There have been no direct security impacts identified with this approach. The
installer should ensure that security policy on the network as whole prevents
IP spoofing which could divert legitimate traffic to an unintended host. This
is a concern whether or not the OpenStack nodes are using BGP themselves, and
may be an issue in environments using traditional routing architecture or
static routes.


Upgrade Impact
==============

When (if) we remove the capability to manage network resources in the
overcloud heat stack, we will need to evaluate whether we want to continue
to provide BGP configuration as a part of the overcloud configuration.

If an operator wishes to begin using BGP routing at the same time as
upgrading the version of OpenStack used they will need to provide the
required configuration parameters if they differ from the defaults provided
in the TripleO deployment service.


Performance Impact
==================

No performance impacts are expected, either positive or negative by using
this approach. Attempts have been made to minimize memory and CPU usage by
using conservative defaults in the configuration.


Documentation Impact
====================

This is a new TripleO deployment service and should be properly documented
to instruct installers in the configuration of FRR for their environment.

The TripleO docs will need updates in many sections, including:

* `TripleO OpenStack Deployment
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/install_overcloud.html>`_
* `Provisioning Baremetal Before Overcloud Deploy
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/baremetal_provision.html#>`_
* `Deploying with Custom Networks
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/custom_networks.html>`_
* `Configuring Network Isolation
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/network_isolation.html>`_
* `Deploying Overcloud with L3 routed networking
  <https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/features/routed_spine_leaf_network.html>`_

The FRR daemons are documented elsewhere, and we should not need to document
usage of BGP in general, as this is a standard protocol. The configuration of
top-of-rack switches is different depending on the make and model of routing
switch used, and we should not expect to provide configuration examples for
network hardware.

Implementation
==============

The implementation will require a new TripleO deployment service, container
definition, and modifications to the existing role definitions. Those changes
are proposed upstream, see the References section for URL links.


Assignee(s)
===========

Primary assignee:
  * Dan Sneddon

Secondary assignees:
  * Michele Baldessari
  * Carlos Gonclaves
  * Daniel Alvarez Sanchez
  * Luis Tomas Bolivar


Work Items
==========

* Develop the container definition
* Define the TripleO deployment service templates
* Define the TripleO Ansible role
* Modify the existing TripleO roles to support the above changes
* Merge the changes to the container, deployment service, and Ansible role
* Ensure FRR packages are available for supported OS versions


References
==========

.. [1] `Review: DNR/DNM Frr support <https://review.opendev.org/c/openstack/tripleo-common/+/763087>`_.
.. [2] `Review: Add tripleo_frr role <https://review.opendev.org/c/openstack/tripleo-ansible/+/763572>`_.
.. [3] `Review: WIP/DNR/DNM FRR service <https://review.opendev.org/c/openstack/tripleo-heat-templates/+/763657>`_.
.. [4] `OVN BGP Agent <https://gist.github.com/luis5tb/93cc01ebfea5d44abf07c0303e7d1514>`_.

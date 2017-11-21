..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
IPSEC encrypted networks
========================

https://blueprints.launchpad.net/tripleo/+spec/ipsec

This proposes the usage of IPSEC tunnels for encrypting all communications in a
TripleO cloud.

Problem Description
===================

Having everything in the network encrypted is a hard requirements for certain
use-cases. While TLS everywhere provides support for this, not everyone wants a
full-fledged CA. IPSEC provides an alternative which requires one component
less (the CA) while still fulfilling the security requirements. With the
downside that IPSEC tunnel configurations can get quite verbose.


Proposed Change
===============

Overview
--------

As mentioned in the mailing list [1], for OSP10 we already worked on an ansible
role that runs on top of a TripleO deployment [2].

It does the following:

* Installs IPSEC if it's not available in the system.

* Sets up the firewall rules.

* Based on a hard-coded set of networks, it discovers the IP addresses for each
  of them.

* Based on a hard-coded set of networks, it discovers the Virtual IP addresses
  (including the Redis VIP).

* It puts up an IPSEC tunnel for most IPs in each network.

  - Regular IPs are handled as a point-to-point IPSEC tunnel.

  - Virtual IPs are handled with road-warrior configurations. This means that
    the VIP's tunnel listens for any connections. This enables easier
    configuration of the tunnel, as the VIP-holder doesn't need to be aware nor
    configure each tunnel.

  - Similarly to TLS everywhere, this focuses on service-to-service
    communication, so we explicitly skip the tenant network. Or,
    as it was in the original ansible role, compute-to-compute communication.
    This significantly reduces the amount of tunnels we need to set up, but
    leaves application security to the deployer.

  - Authentication for the tunnels is done via a Pre-Shared Key (PSK), which is
    shared between all nodes.

* Finally, it creates an OCF resource that tracks each VIP and puts up or down
  its corresponding IPSEC tunnel depending on the VIP's location.

  - While this resource is still in the repository [3], it has now landed
    upstream [4]. Once this resource is available in the packaged version of
    the resource agents, the preferred version will be the packaged one.

  - This resource effectively handles VIP fail-overs, by detecting that a VIP
    is no longer hosted by the node, it cleanly puts down the IPSEC tunnel and
    enables it where the VIP is now hosted.

All of this work is already part of the role, however, to have better
integration with the current state of TripleO, the following work is needed:

* Support for composable networks.

  - Now that composable networks are a thing, we can no longer rely on the
    hard-coded values we had in the role.

  - Fortunately, this is information we can get from the tripleo dynamic
    inventory. So we would need to add information about the available networks
    and the VIPs.

* Configurable skipping of networks.

  - In order to address the tenant network skipping, we need to somehow make it
    configurable.

* Add the IPSEC package as part of the image.

* Configure Firewall rules the TripleO way.

  - Currently the role handles the firewall rule setup. However, it should be
    fairly simple to configure these rules the same way other services
    configure theirs (Using the tripleo.<service>.firewall_rules entry). This
    will require the usage of a composable service template.

* As mentioned above, we will need to create a composable service template.

  - This could take into use the recently added `external_deploy_tasks` section
    of the templates, which will work similarly to the Kubernetes configuration
    and would rely on the config-download mechanism [5].

Alternatives
------------

While deployers can already use TLS everywhere. A few are already using the
aforementioned ansible role. So this would provide a seamless upgrade path for
them.

Security Impact
---------------

This by itself is a security enhancement, as it enables encryption in the
network.

The PSK being shared by all the nodes is not ideal and could be addressed by
per-network PSKs. However, this work could be done in further iterations.

Other End User Impact
---------------------

Currently, the deployer needs to provide their PSK. However, this could be
automated as part of the tasks that TripleO does.

Performance Impact
------------------

Same as with TLS everywhere, adding encryption in the network will have a
performance impact. We currently don't have concrete data on what this impact
actually is.

Other Deployer Impact
---------------------

This would be added as a composable service. So it would be something that the
deployer would need to enable via an environment file.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jaosorior

Work Items
----------

* Add libreswan (IPSEC's frontend) package to the overcloud-full iamge.

* Add required information to the dynamic inventory (networks and VIPs)

* Based on the inventory, create the IPSEC tunnels dynamically, and not based
  on the hardcoded networks.

* Add tripleo-ipsec ansible role as part of the TripleO umbrella.

* Create composable service.


Dependencies
============

* This requires the triple-ipsec role to be available. For this, it will be
  moved to the TripleO umbrella and packaged as such.


Testing
=======

Given that this doesn't require an extra component, we could test this as part
of our upstream tests. The requirement being that the deployment has
network-isolation enabled.


References
==========

[1] http://lists.openstack.org/pipermail/openstack-dev/2017-November/124615.html
[2] https://github.com/JAORMX/tripleo-ipsec
[3] https://github.com/JAORMX/tripleo-ipsec/blob/master/files/ipsec-resource-agent.sh
[4] https://github.com/ClusterLabs/resource-agents/blob/master/heartbeat/ipsec
[5] https://github.com/openstack/tripleo-heat-templates/blob/master/extraconfig/services/kubernetes-master.yaml#L58

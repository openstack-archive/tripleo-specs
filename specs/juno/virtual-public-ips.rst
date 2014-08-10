..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Virtual IPs for public addresses
================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/tripleo/+specs/tripleo-juno-virtual-public-ips

The current public IP feature is intended to specify the endpoint that a cloud
can be reached at. This is typically something where HA is highly desirable.

Making the public IP be a virtual IP instead of locally bound to a single
machine should increase the availability of the clustered service, once we
increase the control plane scale to more than one machine.

Problem Description
===================

Today, we run all OpenStack services with listening ports on one virtual IP.

This means that we're exposing RabbitMQ, MySQL and possibly other cluster-only
services to the world, when really what we want is public services exposed to
the world and cluster only servers not exposed to the world. Deployers are
(rightfully) not exposing our all-services VIP to the world, which leads to
them having to choose between a) no support for externally visible endpoints,
b) all services attackable or c) manually tracking the involved ports and
playing a catch-up game as we evolve things.

Proposed Change
===============

Create a second virtual IP from a user supplied network. Bind additional copies
of API endpoints that should be publically accessible to that virtual IP. We
need to keep presenting them internally as well (still via haproxy and the
control virtual IP) so that servers without any public connectivity such as
hypervisors can still use the APIs (though they may need to override the IP to
use in their hosts files - we have facilities for that already).

The second virtual IP could in principle be on a dedicated ethernet card, or
on a VLAN on a shared card. For now, lets require the admin to specify the
interface on which keepalived should be provisioning the shared IP - be that
``br-ctlplane``, ``vlan25`` or ``eth2``. Because the network topology may be
independent, the keepalive quorum checks need to take place on the specified
interface even though this costs external IP addresses.

The user must be able to specify the same undercloud network as they do today
so that small installs are not made impossible - requiring two distinct
networks is likely hard for small organisations. Using the same network would
not imply using the same IP address - a dedicated IP address will still be
useful to permit better testing confidence and also allows for simple exterior
firewalling of the cluster.

Alternatives
------------

We could not do HA for the public endpoints - not really an option.

We could not do public endpoints and instead document how to provide border
gateway firewalling and NAT through to the endpoints. This just shifts the
problem onto infrastructure we are not deploying, making it harder to deploy.

Security Impact
---------------

Our security story improves by making this change, as we can potentially
start firewalling the intra-cluster virtual IP to only allow known nodes to
connect. Short of that, our security story has improved since we started
binding to specific ips only, as that made opening a new IP address not
actually expose core services (other than ssh) on it.

Other End User Impact
---------------------

End users will need to be able to find out about the new virtual IP. That
should be straight forward via our existing mechanisms.

Performance Impact
------------------

None anticipated.

Other Deployer Impact
---------------------

Deployers will require an additional IP address either on their undercloud
ctlplane network (small installs) or on their public network (larger/production
installs).

Developer Impact
----------------

None expected.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  lifeless (hahahaha)

Other contributors:
  None.

Work Items
----------

* Generalise keepalived.conf to support multiple VRRP interfaces.

* Add support for binding multiple IPs to the haproxy configuration.

* Add logic to incubator and/or heat templates to request a second virtual IP.

* Change heat templates to bind public services to the public virtual IP.

* Possibly tweak setup-endpoints to cooperate, though the prior support
  should be sufficient.

These are out of scope for this, but necessary to use it - I intend to put
them in the discussion in Dan's network overhaul spec.

* Add optional support to our heat templates to boot the machines with two
  nics, not just one - so that we have an IP address for the public interface
  when its a physical interface. We may find there are ordering / enumeration
  issues in Nova/Ironic/Neutron to solve here.

* Add optional support to our heat templates for statically allocating a port
  from neutron and passing it into the control plane for when we're using
  VLANs.

Dependencies
============

None.

Testing
=======

This will be on by default, so our default CI path will exercise it.

Additionally we'll be using it in the up coming VLAN test job which will
give us confidence it works when the networks are partitoned.

Documentation Impact
====================

Add to the manual is the main thing.

References
==========

None

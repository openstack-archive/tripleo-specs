..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
External Load Balancer
======================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-mitaka-external-load-balancer

Make it possible to use (optionally) an external load balancer as frontend for
the Overcloud.


Problem Description
===================

To use an external load balancer the Overcloud templates and manifests will be
updated to accomplish the following three:

* accept a list of virtual IPs as parameter to be used instead of the virtual
  IPs which are normally created as Neutron ports and hosted by the controllers

* make the deployment and configuration of HAProxy on the controllers optional

* allow for the assignment of a predefined list of IPs to the controller nodes
  so that these can be used for the external load balancer configuration


Proposed Change
===============

Overview
--------

The VipMap structure, governed by the ``OS::TripleO::Network::Ports::NetIpMap``
resource type, will be switched to ``OS::TripleO::Network::Ports::NetVipMap``,
a more specific resource type so that it can pointed to a custom YAML allowing
for the VIPs to be provided by the user at deployment time. Any reference to the
VIPs in the templates will be updated to gather the VIP details from such a
structure. The existing VIP resources will also be switched from the non
specialized type ``OS::TripleO::Controller::Ports::InternalApiPort`` into a
more specific type ``OS::TripleO::Network::Ports::InternalApiVipPort`` so that
it will be possible to noop the VIPs or add support for more parameters as
required and independently from the controller ports resource.

The deployment and configuration of HAProxy on the controller nodes will become
optional and driven by a new template parameter visible only to the controllers.

It will be possible to provide via template parameters a predefined list of IPs
to be assigned to the controller nodes, on each network, so that these can be
configured as target IPs in the external load balancer, before the deployment
of the Overcloud is initiated. A new port YAML will be provided for the purpose;
when using an external load balancer this will be used for resources like
``OS::TripleO::Controller::Ports::InternalApiPort``.

As a requirement for the deployment process to succeed, the external load
balancer must be configured in advance with the appropriate balancing rules and
target IPs. This is because the deployment process itself uses a number of
infrastructure services (database/messaging) as well as core OpenStack services
(Keystone) during the configuration steps. A validation script will be provided
so that connectivity to the VIPs can be tested in advance and hopefully avoid
false negatives during the deployment.

Alternatives
------------

None.

Security Impact
---------------

By filtering the incoming connections for the controller nodes, an external load
blancer might help the Overcloud survive network flood attacks or issues due
to purposely malformed API requests.

Other End User Impact
---------------------

The deployer wishing to deploy with an external load balancer will have to
provide at deployment time a few more parameters, amongst which:

* the VIPs configured on the balancer to be used by the Overcloud services

* the IPs to be configured on the controllers, for each network

Performance Impact
------------------

Given there won't be any instance of HAProxy running on the controllers, when
using an external load balancer these might benefit from a lower stress on the
TCP stack.

Other Deployer Impact
---------------------

None expected unless deploying with an external load balancer. A sample
environment file will be provided to provide some guidance over the parameters
to be passed when deploying with an external load balancer.

Developer Impact
----------------

In those scenarios where the deployer was using only a subset of the isolated
networks, the customization templates will need to be updated so that the new
VIPs resource type is nooped. This can be achieved with something like:

.. code::

  resource_registry:
    OS::TripleO::Network::Ports::InternalApiVipPort: /path/to/network/ports/noop.yaml


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gfidente

Other contributors:
  dprince

Work Items
----------

* accept user provided collection of VIPs as parameter

* make the deployment of the managed HAProxy optional

* allow for the assignment of a predefined list of IPs to the controller nodes

* add a validation script to test connectivity against the external VIPs


Dependencies
============

None.


Testing
=======

The feature seems untestable in CI at the moment but it will be possible to test
at least the assignment of a predefined list of IPs to the controller nodes by
providing only the predefined list of IPs as parameter.


Documentation Impact
====================

In addition to documenting the specific template parameters needed when
deploying with an external load balancer, it will also be necessary to provide
some guidance for the configuration of the load balancer configuration so that
it will behave as expected in the event of a failure. Unfortunately the
configuration settings are strictly dependent on the balancer in use; we should
publish a copy of a managed HAProxy instance config to use as reference so that
a deployer could configure his external appliance similarily.


References
==========

None.

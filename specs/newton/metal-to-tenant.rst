..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Metal to Tenant: Ironic in Overcloud
====================================

https://blueprints.launchpad.net/tripleo/+spec/ironic-integration

This blueprint adds support for providing bare metal machines to tenants by
integrating Ironic to the overcloud.


Problem Description
===================

There is an increasing interest in providing bare metal machines to tenants in
the overcloud in addition to or instead of virtual instances. One example is
Sahara: users hope to achieve better performance by removing the hypervisor
abstraction layer in order to eliminate the noisy neighbor effect. For that
purpose, the OpenStack Bare metal service (Ironic) provides an API and a Nova
driver to serve bare metal instances behind the same Nova and Neutron API's.
Currently however TripleO does not support installing and configuring Ironic
and Nova to serve bare metal instances to the tenant.


Proposed Change
===============

Composable Services
-------------------

In the bare metal deployment case, the nova-compute service is only a thin
abstraction layer around the Ironic API. The actual compute instances in
this case are the bare metal nodes. Thus a TripleO deployment with support for
only bare metal nodes will not need dedicated compute nodes in the overcloud.
The overcloud nova-compute service will therefore be placed on controller nodes.

New TripleO composable services will be created and optionally deployed on the
controller nodes:

* ``OS::TripleO::Services::IronicApi`` will deploy the bare metal API.

* ``OS::TripleO::Services::IronicNovaCompute`` will deploy nova compute
  with Ironic as a back end. It will also configure the nova compute to use
  `ClusteredComputeManager
  <https://github.com/openstack/ironic/blob/master/ironic/nova/compute/manager.py>`_
  provide by Ironic to work around inability to have several nova compute
  instances configured with Ironic.

* ``OS::TripleO::Services::IronicConductor`` will deploy a TFTP server,
  an HTTP server (for an optional iPXE environment) and an ironic-conductor
  instance. The ironic-conductor instance will not be managed by pacemaker
  in the HA scenario, as  Ironic has its own Active/Active HA model,
  which spreads load on all active conductors using a hash ring.

  There is no public data on how many bare metal nodes each conductor
  can handle, but the Ironic team expects an order of hundreds of nodes
  per conductor.

Since this feature is not a requirement in all deployments, this will be
opt-in by having a separate environment file.

Hybrid Deployments
------------------

For hybrid deployments with both virtual and bare metal instances, we will use
Nova host aggregates: one for all bare metal hosts, the other for all virtual
compute nodes. This will prevent virtual instances being deployed on baremetal
nodes. Note that every bare metal machine is presented as a separate
Nova compute host. These host aggregates will always be created, even for
purely bare metal deployments, as users might want to add virtual computes
later.

Networking
----------

As of Mitaka, Ironic only supports flat networking for all tenants and for
provisioning. The **recommended** deployment layout will consist of two networks:

* The ``provisioning`` / ``tenant`` network. It must have access to the
  overcloud Neutron service for DHCP, and to overcloud baremetal-conductors
  for provisioning.

  .. note:: While this network can technically be the same as the undercloud
            provisioning network, it's not recommended to do so due to
            potential conflicts between various DHCP servers provided by
            Neutron (and in the future by ironic-inspector).

* The ``management`` network. It will contain the BMCs of bare metal nodes,
  and it only needs access to baremetal-conductors. No tenant access will be
  provided to this network.

  .. note:: Splitting away this network is not really required if tenants are
            trusted (which is assumed in this spec) and BMC access is
            reasonably restricted.

Limitations
-----------

To limit the scope of this spec the following definitely useful features are
explicitly left out for now:

* ``provision`` <-> ``tenant`` network separation (not yet implemented by
  ironic)

* in-band inspection (requires ironic-inspector, which is not yet HA-ready)

* untrusted tenants (requires configuring secure boot and checking firmwares,
  which is vendor-dependent)

* node autodiscovery (depends on ironic-inspector)

Alternatives
------------

Alternatively, we could leave configuring a metal-to-tenant environment up to
the operator.

We could also have it enabled by default, but most likely it won't be required
in most deployments.

Security Impact
---------------

Most of the security implications have to be handled within Ironic. Eg. wiping
the hard disk, checking firmwares, etc. Ironic needs to be configured to be
able to run these jobs by enabling automatic cleaning during node lifecycle.
It is also worth mentioning that we will assume trusted tenants for these bare
metal machines.

Other End User Impact
---------------------

The ability to deploy Ironic in the overcloud will be optional.

Performance Impact
------------------

If enabled, TripleO will deploy additional services to the overcloud:

* ironic-conductor

* a TFTP server

* an HTTP server

None of these should have heavy performance requirements.

Other Deployer Impact
---------------------

None.

Developer Impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ifarkas

Other contributors:
  dtantsur, lucasagomes, mgould, mkovacik

Work Items
----------

when the environment file is included, make sure:

* ironic is deployed on baremetal-conductor nodes

* nova compute is deployed and correctly configured, including:

  * configuring Ironic as a virt driver

  * configuring ClusteredComputeManager

  * setting ram_allocation_ratio to 1.0

* host aggregates are created

* update documentation


Dependencies
============

None.


Testing
=======

This is testable in the CI with nested virtualization and tests will be added
to the tripleo-ci jobs.


Documentation Impact
====================

* Quick start documentation and a sample environment file will be provided.

* Document how to enroll new nodes in overcloud ironic (including host
  aggregates)


References
==========

* `Host aggregates <https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/4/html/Configuration_Reference_Guide/host-aggregates.html>`_

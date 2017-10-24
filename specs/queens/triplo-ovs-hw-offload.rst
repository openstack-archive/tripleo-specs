..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Adding OVS Hardware Offload to TripleO
==========================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-ovs-hw-offload

OVS Hardware Offload leverages SR-IOV technology to control the SR-IOV
VF using VF representor port. OVS 2.8.0 supports the hw-offload option which
allows to offload OVS datapath rule to hardware using linux traffic control
tool and the VF representor port. This feature accelerates the OVS
with a SR-IOV NIC which support switchdev mode.

Problem Description
===================

Today the installation and configuration of OVS hardware offload feature is
done manually after overcloud deployment. It shall be automated via tripleo.

Proposed Change
===============

Overview
--------

* Configure the SR-IOV NIC to be in switchdev mode using the following
  syntax <physical_interface>:<numvfs>:<mode> for NeutronSriovNumVFs.
  mode can be legacy or switchdev
* Configure the OVS with other_config:hw-offload. The options can
  be added for the cluster without side effects, because if then NIC doesn't
  support OVS will fall-back to kernel datapath.

* Nova scheduler should be configured to use the PciPassthroughFilter
  (same SR-IOV)
* Nova compute should be configured with passthrough_whitelist (same SR-IOV)

Alternatives
------------

None

Security Impact
---------------

None

Other End User Impact
---------------------

None

Performance Impact
------------------

* OVS Hardware Offload leverage the SR-IOV technology to provides near
  native I/O performance for each virtual machine that managed by OpenVswitch.

Other Deployer Impact
---------------------

* The operator shall ensure that the BIOS supports VT-d/IOMMU virtualization
  technology on the compute nodes.

* IOMMU needs to be enabled in the Compute+SR-IOV nodes. Boot parameters
  (intel_iommu=on or  amd_iommu=pt) shall be added in the grub.conf, using the
  PreNetworkConfig.

* Post deployment, operator shall

  * Create neutron ports prior to creating VM’s (nova boot)
    openstack port create --vnic-type direct --binding-profile '{"capabilities": ["switchdev"]}' port1

  * Create the VM with the required flavor and SR-IOV port id
    openstack server create --image cirros-mellanox_sriov --port=port1 --flavor m1.tiny vm_a1

Developer Impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  waleedm (waleedm@mellanox.com

Other contributors:
  moshele (moshele@mellanox.com)

Work Items
----------

* Update tripleo::host::sriov::numvfs_persistence to allow configure SR-IOV
  in switchdev mode. extending the vf_defs to
  <physical_interface>:<numvfs>:<mode>. Mode can be legacy which is default
  SR-IOV or switchdev which is used for ovs hardware offload.
* Add a template parameter called NeutronOVSHwOffload to enable.
* provide environment YAML for OVS hardware offload in tripleo-heat-templates.

Dependencies
============

None


Testing
=======

* Since SR-IOV needs specific hardware support, this feature can be tested
  under third party CI. We hope to provide Mellanox CI to SR-IOV and this
  feature.

Documentation Impact
====================

None

References
==========

* Introduction to SR-IOV
  http://goo.gl/m7jP3

* SR-IOV OVS hardware offload netdevconf
  http://netdevconf.org/1.2/papers/efraim-gerlitz-sriov-ovs-final.pdf

* OVS hardware offload in OpenVswitch
  https://mail.openvswitch.org/pipermail/ovs-dev/2017-April/330606.html

* OpenStack OVS mechanism driver support in neutron/nova/os-vif
  https://review.openstack.org/#/c/398265/
  https://review.openstack.org/#/c/275616/
  https://review.openstack.org/#/c/460278/

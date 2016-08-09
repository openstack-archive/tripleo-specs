
This work is licensed under a Creative Commons Attribution 3.0 Unported
License.

http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Adding OVS-DPDK to Tripleo
==========================================

Blueprint URL -
https://blueprints.launchpad.net/tripleo/+spec/tripleo-ovs-dpdk

DPDK is a set of libraries and drivers for fast packet processing and gets as
close to wire-line speed as possible for virtual machines.

  * It is a complete framework for fast packet processing in data plane
    applications.

  * Directly polls the data from the NIC.

  * Does not use interrupts - to prevent performance overheads.

  * Uses the hugepages to preallocate large regions of memory, which allows the
    applications to DMA data directly into these pages.

  * DPDK also has its own buffer and ring management systems for handling
    sk_buffs efficiently.

DPDK provides data plane libraries and NIC drivers for -

  * Queue management with lockless queues.

  * Buffer manager with pre-allocated fixed size buffers.

  * PMD (poll mode drivers) to work without asynchronous notifications.

  * Packet framework (set of libraries) to help data plane packet processing.

  * Memory manager - allocates pools of memory, uses a ring to store free
    objects.

Problem Description
===================

* Today the installation and configuration of OVS+DPDK in openstack is done
  manually after overcloud deployment. This can be very challenging for the
  operator and tedious to do over a large number of compute nodes.
  The installation of OVS+DPDK needs be automated in tripleo.

* Identification of the hardware capabilities for DPDK were all done manually
  today and the same shall be automated during introspection. This hardware
  detection also provides the operator with the data needed for configuring
  Heat templates.

* As of today its not possible to have the co-existence of compute nodes with
  DPDK enabled hardware and without DPDK enabled hardware.


Proposed Change
===============

* Ironic Python Agent shall discover the below hardware details and store it
  in swift blob -

  * CPU flags for hugepages support -
    If pse exists then 2MB hugepages are supported
    If pdpe1gb exists then 1GB hugepages are supported

  * CPU flags for IOMMU -
    If VT-d/svm exists, then IOMMU is supported, provided IOMMU support is
    enabled in BIOS.

  * Compatible nics -
    Shall compare it with the list of NICs whitelisted for DPDK. The DPDK
    supported NICs are available at http://dpdk.org/doc/nics

  The nodes without any of the above mentioned capabilities can't be used for
  COMPUTE role with DPDK.

* Operator shall have a provision to enable DPDK on compute nodes

* The overcloud image for the nodes identified to be COMPUTE capable and having
  DPDK NICs, shall have the OVS+DPDK package instead of OVS. It shall also have
  packages dpdk and driverctl.

* The device names of the DPDK capable NIC’s shall be obtained from T-H-T.
  The PCI address of DPDK NIC needs to be identified from the device name.
  It is required for whitelisting the DPDK NICs during PCI probe.

* Hugepages needs to be enabled in the Compute nodes with DPDK.
  Bug: https://bugs.launchpad.net/tripleo/+bug/1589929 needs to be implemeted

* CPU isolation needs to be done so that the CPU cores reserved for DPDK Poll
  Mode Drivers (PMD) are not used by the general kernel balancing,
  interrupt handling and scheduling algorithms.
  Bug: https://bugs.launchpad.net/tripleo/+bug/1589930 needs to be implemented.

* On each COMPUTE node with DPDK enabled NIC, puppet shall configure the
  DPDK_OPTIONS for whitelisted NIC's, CPU mask and number of memory channels
  for DPDK PMD. The DPDK_OPTIONS needs to be set in /etc/sysconfig/openvswitch

* Os-net-config shall -

  * Associate the given interfaces with the dpdk drivers (default as vfio-pci
    driver) by identifying the pci address of the given interface. The
    driverctl shall be used to bind the driver persistently

  * Understand the ovs_user_bridge and ovs_dpdk_port types and configure the
    ifcfg scripts accordingly.

  * The "TYPE" ovs_user_bridge shall translate to OVS type OVSUserBridge and
    based on this OVS will configure the datapath type to 'netdev'.

  * The "TYPE" ovs_dpdk_port shall translate OVS type OVSDPDKPort and based on
    this OVS adds the port to the bridge with interface type as 'dpdk'

  * Understand the ovs_dpdk_bond and configure the ifcfg scripts accordingly.

* On each COMPUTE node with DPDK enabled NIC, puppet shall -

  * Enable OVS+DPDK in /etc/neutron/plugins/ml2/openvswitch_agent.ini
    [OVS]
    datapath_type=netdev
    vhostuser_socket_dir=/var/run/openvswitch

  * Configure vhostuser ports in /var/run/openvswitch to be owned by qemu.

* On each controller node, puppet shall -

  * Add NUMATopologyFilter to scheduler_default_filters in nova.conf.

Alternatives
------------

* The boot parameters could be configured via puppet (during overcloud
  deployment) as well as virt-customize (after image building or downloading).
  The choice of selection of boot parameter is moved out of scope of this
  blueprint and would be tracked via
  https://bugs.launchpad.net/tripleo/+bug/1589930.

Security impact
---------------

* We have no firewall drivers which support ovs-dpdk at present. Security group
  support with conntrack is a possible option, and this work is in progress.
  Security groups will not be supported.


Other End User Impact
---------------------

None

Performance Impact
------------------

* OVS-DPDK can augment 3 times dataplane performance.
  Refer http://goo.gl/Du1EX2

Other Deployer Impact
---------------------

* The operator shall ensure that the VT-d/IOMMU virtualization technology is
  enabled in BIOS of the compute nodes.

* Post deployment, operator shall modify the VM flavors for using hugepages,
  CPU pinning
  Ex: nova flavor-key m1.small set "hw:mem_page_size=large"


Developer Impact
----------------

None

Implementation
==============


Assignee(s)
-----------

Primary assignees:

* karthiks
* sanjayu

Work Items
----------

* The proposed changes discussed earlier will be the work items

Dependencies
============

* We are dependent on composable roles, as this is something we would
  require only on specific compute nodes and not generally on all the nodes.

* To enable Hugepages, bug: https://bugs.launchpad.net/tripleo/+bug/1589929
  needs to be implemeted

* To address boot parameter changes for CPU isolation,
  bug: https://bugs.launchpad.net/tripleo/+bug/1589930 needs to be implemented

Testing
=======

* Since DPDK needs specific hardware support, this feature cant be tested under
  CI. We will need third party CI for validating it.

Documentation Impact
====================

* Manual steps that needs to be done by the operator shall be documented.
  Ex: configuring BIOS for VT-d, adding boot parameter for CPU isolation,
  hugepages, post deploymenent configurations.

Refrences
=========

* Manual steps to setup DPDK in RedHat Openstack Platform 8
  https://goo.gl/6ymmJI

* Setup procedure for CPU pinning and NUMA topology
  http://goo.gl/TXxuhv

* DPDK supported NICS
  http://dpdk.org/doc/nics




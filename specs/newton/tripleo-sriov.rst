This work is licensed under a Creative Commons Attribution 3.0 Unported
License.

http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Adding SR-IOV to Tripleo
==========================================

Blueprint URL:
  https://blueprints.launchpad.net/tripleo/+spec/tripleo-sriov

SR-IOV is a specification that extends the PCI Express specification and allows
a PCIe device to appear to be multiple separate physical PCIe devices.

SR-IOV provides one or more Virtual Functions (VFs) and a Physical Function(PF)

  * Virtual Functions (VF's) are ‘lightweight’ PCIe functions that contain the
    resources necessary for data movement but have a carefully minimized set
    of configuration resources.

  * Physical Function (PF) are full PCIe functions that include the SR-IOV
    Extended Capability. This capability is used to configure and manage
    the SR-IOV functionality.

The VF’s could be attached to VMs like a dedicated PCIe device and thereby the
usage of SR-IOV NICs boosts the networking performance considerably.


Problem Description
===================

* Today the installation and configuration of SR-IOV feature is done manually
  after overcloud deployment. It shall be automated via tripleo.

* Identification of the hardware capabilities for SR-IOV were all done manually
  today and the same shall be automated during introspection. The hardware
  detection also provides the operator, the data needed for configuring Heat
  templates.

Proposed Change
===============

Overview
--------

* Ironic Python Agent will discover the below hardware details and store it in
  swift blob -

  * SR-IOV capable NICs:
    Shall read /sys/bus/pci/devices/.../sriov_totalvfs and check if its non
    zero, inorder to identify if the NIC is SR-IOV capable

  * VT-d or IOMMU support in BIOS:
    The CPU flags shall be read to identify the support.

* DIB shall include the package by default - openstack-neutron-sriov-nic-agent.

* The nodes without any of the above mentioned capabilities can't be used for
  COMPUTE role with SR-IOV

* SR-IOV drivers shall be loaded during bootup via persistent module loading
  scripts. These persistent module loading scripts shall be created by the
  puppet manifests.

* T-H-T shall provide the below details

  * supported_pci_vendor_devs - configure the vendor-id/product-id couples in
    the nodes running neutron-server

  * max number of vf's - persistent across reboots

  * physical device mappings - Add physical device mappings ml2_conf_sriov.ini
    file in compute node

* On the nodes running the Neutron server, puppet shall

  * enable sriovnicswitch in the /etc/neutron/plugin.ini file
    mechanism_drivers = openvswitch,sriovnicswitch
    This configuration enables the SR-IOV mechanism driver alongside
    OpenvSwitch.

  * Set the VLAN range for SR-IOV in the file /etc/neutron/plugin.ini, present
    in the network node
    network_vlan_ranges = <physical network name SR-IOV interface>:<VLAN min>
    :<VLAN max> Ex :  network_vlan_ranges = fabric0:15:20

  * Configure the vendor-id/product-id couples if it differs from
    “15b3:1004,8086:10ca” in /etc/neutron/plugins/ml2/ml2_conf_sriov.ini
    supported_pci_vendor_devs = 15b3:1004,8086:10ca,<vendor-id:product-id>

  * Configure neutron-server.service to use the ml2_conf_sriov.ini file
    [Service] Type=notify User=neutron ExecStart=/usr/bin/neutron-server
    --config-file /usr/share/neutron/neutron-dist.conf --config-file
    /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini
    --config-file /etc/neutron/plugins/ml2/ml2_conf_sriov.ini  --log-file
    /var/log/neutron/server.log

* In the nodes running nova scheduler, puppet shall

  * add PciPassthroughFilter filter to the list of scheduler_default_filters.
    This needs to be done to allow proper scheduling of SR-IOV devices

* On each COMPUTE+SRIOV node, puppet shall configure /etc/nova/nova.conf

  * Associate the available VFs with each physical network
    Ex: pci_passthrough_whitelist={"devname": "enp5s0f1",
    "physical_network":"fabric0"}

    PCI passthrough whitelist entries use the following syntax: ["device_id":
    "<id>",] ["product_id": "<id>",] ["address":
    "[[[[<domain>]:]<bus>]:][<slot>][.[<function>]]" | "devname": "Ethernet
    Interface Name",] "physical_network":"Network label string"

    VF's that needs to be excluded from agent configuration shall be added in
    [sriov_nic]/exclude_devices. T-H-T shall configure this.

    Multiple whitelist entries per host are supported.

* Puppet shall

  * Setup max number of VF's to be configured by the operator
    echo required_max_vfs > /sys/bus/pci/devices/.../sriov_numvfs
    Puppet will also validate the required_max_vfs, so that it does not go
    beyond the supported max on the device.

  * Enable NoopFirewallDriver in the
    '/etc/neutron/plugins/ml2/sriov_agent.ini' file.

    [securitygroup]
    firewall_driver = neutron.agent.firewall.NoopFirewallDriver

  * Add mappings to the /etc/neutron/plugins/ml2/sriov_agent.ini file.  Ex:
    physical_device_mappings = fabric0:enp4s0f1
    In this example, fabric0 is the physical network, and enp4s0f1 is the
    physical function

* Puppet shall start the SR-IOV agent on Compute

  * systemctl enable  neutron-sriov-nic-agent.service

  * systemctl start neutron-sriov-nic-agent.service


Alternatives
------------

None

Security impact
---------------

* We have no firewall drivers which support SR-IOV at present.
  Security groups will be disabled only for SR-IOV ports in compute hosts.


Other End User Impact
---------------------

None

Performance Impact
------------------

* SR-IOV provides near native I/O performance for each virtual machine on a
  physical server. Refer - http://goo.gl/HxZvXX


Other Deployer Impact
---------------------

* The operator shall ensure that the BIOS supports VT-d/IOMMU virtualization
  technology on the compute nodes.

* IOMMU needs to be enabled in the Compute+SR-IOV nodes. Boot parameters
  (intel_iommu=on or  amd_iommu=pt) shall be added in the grub.conf, using the
  first boot scripts (THT).

* Post deployment, operator shall

  * Create neutron ports prior to creating VM’s (nova boot)
    neutron port-create fabric0_0 --name sr-iov --binding:vnic-type direct

  * Create the VM with the required flavor and SR-IOV port id
    Ex: nova boot --flavor m1.small --image <image id> --nic port-id=<port id>
    vnf0

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

* Documented above in the Proposed changes


Dependencies
============

* We are dependent on composable roles as SR-IOV specific changes is something
  we would require on specific compute nodes and not generally on all the
  nodes. Blueprint -
  https://blueprints.launchpad.net/tripleo/+spec/composable-services-within-roles

Testing
=======

* Since SR-IOV needs specific hardware support, this feature cant be tested
  under CI. We will need third party CI for validating it.

Documentation Impact
====================

* Manual steps that needs to be done by the operator shall be documented.
  Ex: configuring BIOS for VT-d, IOMMU, post deploymenent configurations.

Refrences
=========

* SR-IOV support for virtual networking
  https://goo.gl/eKP1oO

* Enable SR-IOV functionality available in OpenStack
  http://docs.openstack.org/liberty/networking-guide/adv_config_sriov.html

* Introduction to SR-IOV
  http://goo.gl/m7jP3

* Setup procedure for CPU pinning and NUMA topology
  http://goo.gl/TXxuhv

* /sys/bus/pci/devices/.../sriov_totalvfs - This file appears when a physical
  PCIe device supports SR-IOV.
  https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci


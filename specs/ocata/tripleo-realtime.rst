..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Add real-time compute nodes to TripleO
======================================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-realtime

Real-time guest VMs require compute nodes with a specific configuration to
control the sources of latency spikes.

Problem Description
===================

Manual configuration of compute nodes to support real-time guests is possible.
However this is complex and time consuming where there is large number of
compute nodes to configure.

On a real-time compute node a subset of the available physical CPUs (pCPUs) are
isolated and dedicated to real-time tasks. The remaining pCPUs are dedicated to
general housekeeping tasks. This requires a real-time Linux Kernel and real-time
KVM that allow their housekeeping tasks to be isolated. The real-time and
housekeeping pCPUs typically reside on different NUMA nodes.

Huge pages are also reserved for guest VMs to prevent page faults, either via
the kernel command line or via sysfs. Sysfs is preferable as it allows the
reservation on each individual NUMA node to be set.

A real-time Linux guest VM is partitioned in a similar manner, having one or
more real-time virtual CPUs (vCPUs) and one or more general vCPUs to handle
the non real-time housekeeping tasks.

A real-time vCPU is pinned to a real-time pCPU while a housekeeping vCPU is
pinned to a housekeeping pCPUS.

It is expected that operators would require both real-time and non real-time
compute nodes on the same overcloud.

Use Cases
---------

The primary use-case is NFV appliances deployed by the telco community which
require strict latency guarantees. Other latency sensitive applications should
also benefit.

Proposed Change
===============

This spec proposes changes to automate the deployment of real-time capable
compute nodes using TripleO.

* a custom overcloud image for the real-time compute nodes, which shall include:

  * real-time Linux Kernel
  * real-time KVM
  * real-time tuned profiles

* a new real-time compute role that is a variant of the exising compute role

  * huge pages shall be enabled on the real-time compute nodes.
  * huge pages shall be reserved for the real-time guests.
  * CPU pinning shall be used to isolate kernel housekeeping tasks from the
    real-time tasks by configuring tuned.
  * CPU pinning shall be used to isolate virtualization housekeeping tasks from
    the real-time tasks by configuring nova.

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

Worse-case latency in real-time guest VMs should be significantly reduced.
However a real-time configuration potentially reduces the overall throughput of
a compute node.

Other Deployer Impact
---------------------

The operator will remain responsible for:

* appropriate BIOS settings on compute node.
* setting appropriate parameters for the real-time role in an environment file
* post-deployment configuration

  * creating/modifying overcloud flavors to enable CPU pinning, hugepages,
    dedicated CPUs, real-time policy
  * creating host aggregates for real-time and non real-time compute nodes



Developer Impact
----------------

None

Implementation
==============

Real-time ``overcloud-full`` image creation:

* create a disk-image-builder element to include the real-time packages
* add support for multiple overcloud images in python-tripleoclient CLIs::

    openstack overcloud image build
    openstack overcloud image upload

Real-time compute role:

* create a ``ComputeRealtime`` role

  * variant of the ``Compute`` role that can be configued and scaled
    independently
  * allows a different image and flavor to be used for real-time nodes
  * includes any additional parameters/resources that apply to real-time nodes

* create a ``NovaRealtime`` service

  * contains a nested ``NovaCompute`` service
  * allows parameters to be overridden for the real-time role only

Nova configuration:

* Nova ``vcpu_pin_set`` support is already implemented. See NovaVcpuPinSet in
  :ref:`references`

Kernel/system configuration:

* hugepages support

  * set default hugepage size (kernel cmdline)
  * number of hugepages of each size to reserve at boot (kernel cmdline)
  * number of hugepages of each size to reserve post boot on each NUMA node
    (sysfs)

* Kernel CPU pinning

  * isolcpu option (kernel cmdline)

Ideally this can be implemented outside of TripleO in the Tuned profiles, where
it is possible to set the kernel command line and manage sysfs. TripleO would
then manage the Tuned profile config files.
Alternatively the grub and systemd config files can be managed directly.

.. note::

  This requirement is shared with OVS-DPDK. The development should be
  coordinated to ensure a single implementation is implemented for
  both use-cases.
  Managing the grub config via a UserData script is the current approach used
  for OVS-DPDK. See OVS-DPDK documentation in :ref:`references`.

Assignee(s)
-----------

Primary assignee:
  owalsh

Other contributors:
  ansiwen

Work Items
----------

As outlined in the proposed changes.

Dependencies
============

* Libvirt real time instances
  https://blueprints.launchpad.net/nova/+spec/libvirt-real-time
* Hugepages enabled in the Compute nodes.
  https://bugs.launchpad.net/tripleo/+bug/1589929
* CPU isolation of real-time and non real-time tasks.
  https://bugs.launchpad.net/tripleo/+bug/1589930
* Tuned
  https://fedorahosted.org/tuned/

Testing
=======

Genuine real-time guests are unlikely to be testable in CI:

* specific BIOS settings are required.
* images with real-time Kernel and KVM modules are required

However the workflow to deploy these guest should be testable in CI.

Documentation Impact
====================

Manual steps performed by the operator shall be documented:

* BIOS settings for low latency
* Real-time overcloud image creation

  .. note::

    CentOS repos do not include RT packages. The CERN CentOS RT repository is an
    alternative.
* Flavor and profile creation
* Parameters required in a TripleO environment file
* Post-deployment configuration

.. _references:

References
==========

Nova blueprint `"Libvirt real time instances"
<https://blueprints.launchpad.net/nova/+spec/libvirt-real-time>`_

The requirements are similar to :doc:`../newton/tripleo-ovs-dpdk`

CERN CentOS 7 RT repo http://linuxsoft.cern.ch/cern/centos/7/rt/

NoveVcpuPinSet parameter added: https://review.openstack.org/#/c/343770/

OVS-DPDK documentation (work-in-progress): https://review.openstack.org/#/c/395431/

..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Deriving TripleO Parameters
===========================

https://blueprints.launchpad.net/tripleo/+spec/tripleo-derive-parameters

This specification proposes a generic interface for automatically
populating environment files with parameters which were derived from
formulas; where the formula's input came from introspected hardware
data, workload type, and deployment type. It also provides specific
examples of how this interface may be used to improve deployment of
overclouds to be used in DPDK or HCI usecases. Finally, it proposes
how this generic interface may be shared and extended by operators
who optionally chose to have certain parameters prescribed so that
future systems tuning expertise may be integrated into TripleO.

Problem Description
===================

Operators must populate parameters for a deployment which may be
specific to hardware and deployment type. The hardware information
of a node is available to the operator once the introspection of the
node is completed. However, the current process requires that the
operator manually read the introspected data, make decisions based on
that data and then update the parameters in an environment file. This
makes deployment preparation unnecessarily complex.

For example, when deploying for DPDK, the operator must provide the
list of CPUs which should be assigned to the DPDK Poll Mode Driver
(PMD) and the CPUs should be provided from the same NUMA node on which
the DPDK interface is present. In order to provide the correct
parameters, the operator must cross check all of these details.

Another example is the deployment of HCI overclouds, which run both
Nova compute and Ceph OSD services on the same nodes. In order to
prevent contention between compute and storage services, the operator
may manually apply formulas, provided by performance tuning experts,
which take into account available hardware, type of workload, and type
of deployment, and then after computing the appropriate parameters
based on those formulas, manually store them in environment files.

In addition to the complexity of the DPDK or HCI usecase, knowing the
process to assign CPUs to the DPDK Poll Mode Driver or isolate compute
and storage resources for HCI is, in itself, another problem. Rather
than document the process and expect operators to follow it, the
process should be captured in a high level language with a generic
interface so that performance tuning experts may easily share new
similar processes for other use cases with operators.

Proposed Change
===============

This spec aims to make three changes to TripleO outlined below.

Mistral Workflows to Derive Parameters
--------------------------------------

A group of Mistral workflows will be added for the features which are
complex to determine the deployment parameters. Features like DPDK,
SR-IOV and HCI require, input from the introspection data to be
analyzed to compute the deployment parameters. This derive parameters
workflow will provide a default set of computational formulas by
analyzing the introspected data. Thus, there will be a hard dependency
with node introspection for this workflow to be successful.

During the first iterations, all the roles in a deployment will be
analyzed to find a service associated with the role, which requires
parameter derivation. Various options of using this and the final
choice for the current iteration is discussed in below section
`Workflow Association with Services`_.

This workflow assumes that all the nodes in a role have a homegenous
hardware specification and introspection data of the first node will
be used for processing the parameters for the entire role. This will
be reexamined in later iterations, based on the need for node specific
derivations. The workflow will consider the flavor-profile association
and nova placement scheduler to identify the nodes associated with a
role.

Role-specific parameters are an important requirement for this workflow.
If there are multiple roles with the same service (feature) enabled,
the parameters which are derived from this workflow will be applied
only on the corresponding role.

The input sources for these workflows are the ironic database and ironic
introspection data stored in Swift, in addition to the Deployment plan stored
in Swift. Computations done to derive the parameters within the Mistral
workflow will be implemented in YAQL. These computations will be a separate
workflow on per feature basis so that the formulas can be customizable. If an
operator has to modify the default formulas, he or she has to update only this
workflow with customized formula.


Applying Derived Parameters to the Overcloud
--------------------------------------------

In order for the resulting parameters to be applied to the overcloud,
the deployment plan, which is stored in Swift on the undercloud,
will be modified with the Mistral `tripleo.parameters.update` action
or similar.

The methods for providing input for derivation and the update of
parameters which are derivation output should be consistent with the
Deployment Plan Management specification [1]_. The implementation of
this spec with respect to the interfaces to set and get parameters may
change as it is updated. However, the basic workflow should remain the
same.

Trigger Mistral Workflows with TripleO
--------------------------------------

Assuming that workflows are in place to derive parameters and update the
deployment plan as described in the previous two sections, an operator may
take advantage of this optional feature by enabling it via ``plan-
environment.yaml``. A new section ``workflow_parameters`` will be added to
the ``plan-environments.yaml`` file to accomodate the additional parameters
required for executing workflows. With this additional section, we can ensure
that the workflow specific parameters are provide only to the workflow,
without polluting the heat environments. It will also be possible to provide
multiple plan environment files which will be merged in the CLI before plan
creation.

These additional parameters will be read by the derive params workflow
directly from the merged ``plan-environment.yaml`` file stored in Swift.

It is possible to modify the created plan or modify the profile-node
association, after the derive parameters workflow execution. As of
now, we assume that there no such alterations done, but it will be
extended after the initial iteration, to fail the deployment with
some validations.

An operator should be able to derive and view parameters without doing a
deployment; e.g. "generate deployment plan". If the calculation is done as
part of the plan creation, it would be possible to preview the calculated
values. Alternatively the workflow could be run independently of the overcloud
deployment, but how that will fit with the UI workflow needs to be determined.

Usecase 1: Derivation of DPDK Parameters
========================================

A part of the Mistral workflow which uses YAQL to derive DPDK
parameters based on introspection data, including NUMA [2]_, exists
and may be seen on GitHub [3]_.

Usecase 2: Derivation Profiles for HCI
======================================

This usecase uses HCI, running Ceph OSD and Nova Compute on the same node. HCI
derive parameters workflow works with a default set of configs to categorize
the type of the workload that the role will host. An option will be provide to
override the default configs with deployment specific configs via ``plan-
environment.yaml``.

In case of HCI deployment, the additional plan environment used for the
deployment will look like::

    workflow_parameters:
      tripleo.workflows.v1.derive_parameters:
        # HCI Derive Parameters
        HciProfile: nfv-default
        HciProfileConfig:
          default:
            average_guest_memory_size_in_mb: 2048
            average_guest_CPU_utilization_percentage: 50
          many_small_vms:
            average_guest_memory_size_in_mb: 1024
            average_guest_CPU_utilization_percentage: 20
          few_large_vms:
            average_guest_memory_size_in_mb: 4096
            average_guest_CPU_utilization_percentage: 80
          nfv_default:
            average_guest_memory_size_in_mb: 8192
            average_guest_CPU_utilization_percentage: 90

In the above example, the section ``workflow_parameters`` is used to provide
input parameters for the workflow in order to isolate Nova and Ceph
resources while maximizing performance for different types of guest
workloads. An example of the derivation done with these inputs is
provided in nova_mem_cpu_calc.py on GitHub [4]_.


Other Integration of Parameter Derivation with TripleO
======================================================

Users may still override parameters
-----------------------------------

If a workflow derives a parameter, e.g. cpu_allocation_ratio, but the
operator specified a cpu_allocation_ratio in their overcloud deploy,
then the operator provided value is given priority over the derived
value. This may be useful in a case where an operator wants all of the
values that were derived but just wants to override a subset of those
parameters.

Handling Cross Dependency Resources
-----------------------------------

It is possible that multiple workflows will end up deriving parameters based
on the same resource (like CPUs). When this happens, it is important to have a
specific order for the workflows to be run considering the priority.

For example, let us consider the resource CPUs and how it should be used
between DPDK and HCI. DPDK requires a set of dedicated CPUs for Poll Mode
Drivers (NeutronDpdkCoreList), which should not be used for host process
(ComputeHostCpusList) and guest VM's (NovaVcpuPinSet). HCI requires the CPU
allocation ratio to be derived based on the number of CPUs that are available
for guest VMs (NovaVcpuPinSet). Priority is given to DPDK, followed by HOST
parameters and then HCI parameters. In this case, the workflow execution
starts with a pool of CPUs, then:

* DPDK: Allocate NeutronDpdkCoreList
* HOST: Allocate ComputeHostCpusList
* HOST: Allocate NovaVcpuPinSet
* HCI: Fix the cpu allocation ratio based on NovaVcpuPinSet

Derived parameters for specific services or roles
-------------------------------------------------

If an operator only wants to configure Enhanced Placement Awareness (EPA)
features like CPU pinning or huge pages, which are not associated with any
feature like DPDK or HCI, then it should be associated with just the compute
service.

Workflow Association with Services
----------------------------------

The optimal way to associate the derived parameter workflows with
services, is to get the list of the enabled services on a given role,
by previewing Heat stack. With the current limitations in Heat, it is
not possible fetch the enabled services list on a role. Thus, a new
parameter will be introduced on the service which is associated with a
derive parameters workflow. If this parameter is referenced in the
heat resource tree, on a specific role, then the corresponding derive
parameter workflow will be invoked. For example, the DPDK service will
have a new parameter "EnableDpdkDerivation" to enable the DPDK
specific workflows.

Future integration with TripleO UI
----------------------------------

If this spec were implemented and merged, then the TripleO UI could
have a menu item for a deployment, e.g. HCI, in which the deployer may
choose a derivation profile and then deploy an overcloud with that
derivation profile.

The UI could better integrate with this feature by allowing a deployer
to use a graphical slider to vary an existing derivation profile and
then save that derivation profile with a new name. The following
cycle could be used by the deployer to tune the overcloud.

* Choose a deployment, e.g. HCI
* Choose an HCI profile, e.g. many_small_vms
* Run the deployment
* Benchmark the planned workload on the deployed overcloud
* Use the sliders to change aspects of the derivation profile
* Update the deployment and re-run the benchmark
* Repeat as needed
* Save the new derivation profile as the one to be deployed in the field

The implementation of this spec would enable the TripleO UI to support
the above.

Alternatives
------------

The simplest alternative is for operators to determine what tunings
are appropriate by testing or reading documentation and then implement
those tunings in the appropriate Heat environment files. For example,
in an HCI scenario, an operator could run nova_mem_cpu_calc.py [4]_
and then create a Heat environment file like the following with its
output and then deploy the overcloud and directly reference this
file::

    parameter_defaults:
      ExtraConfig:
        nova::compute::reserved_host_memory: 75000
	nova::cpu_allocation_ratio: 8.2

This could translate into a variety of overrides which would require
initiative on the operator's part.

Another alternative is to write separate tools which generate the
desired Heat templates but don't integrate them with TripleO. For
example, nova_mem_cpu_calc.py and similar, would produce a set of Heat
environment files as output which the operator would then include
instead of output containing the following:

* nova.conf reserved_host_memory_mb = 75000 MB
* nova.conf cpu_allocation_ratio = 8.214286

When evaluating the above, keep in mind that only two parameters for
CPU allocation and memory are being provided as an example, but that
a tuned deployment may contain more.

Security Impact
---------------

There is no security impact from this change as it sits at a higher
level to automate, via Mistral and Heat, features that already exist.

Other End User Impact
---------------------

Operators need not manually derive the deployment parameters based on the
introspection or hardware specification data, as it is automatically derived
with pre-defined formulas.

Performance Impact
------------------

The deployment and update of an overcloud may take slightly longer if
an operator uses this feature because an additional Mistral workflow
needs to run to perform some analytics before applying configuration
updates. However, the performance of the overcloud would be improved
because this proposal aims to make it easier to tune the overcloud for
performance.

Other Deployer Impact
---------------------

A new configuration option is being added, but it has to be explicitly
enabled, and thus it would not take immediate effect after its merged.
Though, if a deployer chooses to use it and there is a bug in it, then
it could affect the overcloud deployment. If a deployer uses this new
option, and had a deploy in which they set a parameter directly,
e.g. the Nova cpu_allocation_ratio, then that parameter may be
overridden by a particular tuning profile. So that is something a
deployer should be aware of when using this proposed feature.

The config options being added will ship with a variety of defaults
based on deployments put under load in a lab. The main idea is to make
different sets of defaults, which were produced under these
conditions, available. The example discussed in this proposal and to
be made available on completion could be extended.

Developer Impact
----------------

This spec proposes modifying the deployment plan which, if there was a
bug, could introduce problems into a deployment. However, because the
new feature is completely optional, a developer could easily disable
it.

Implementation
==============

Assignee(s)
-----------

Primary assignees:
  skramaja
  fultonj

Other contributors:
  jpalanis
  abishop
  shardy
  gfidente

Work Items
----------

* Derive Params start workflow to find list of roles
* Workflow run for each role to fetch the introspection data and trigger
  individual features workflow
* Workflow to identify if a service associated with a features workflow is
  enabled in a role
* DPDK Workflow: Analysis and concluding the format of the input data (jpalanis)
* DPDK Workflow: Parameter deriving workflow (jpalanis)
* HCI Workflow: Run a workflow that calculates the parameters (abishop)
* SR-IOV Workflow
* EPA Features Workflow
* Run the derive params workflow from CLI
* Add CI scenario testing if workflow with produced expected output

Dependencies
============

* NUMA Topology in introspection data (ironic-python-agent) [5]_

Testing
=======

Create a new scenario in the TripleO CI in which a deployment is done
using all of the available options within a derivation profile called
all-derivation-options. A CI test would need to be added that would
test this new feature by doing the following:

* A deployment would be done with the all-derivation-options profile
* The deployment would be checked that all of the configurations had been made
* If the configuration changes are in place, then the test passed
* Else the test failed

Relating the above to the HCI usecase, the test could verify one of
two options:

1. A Heat environment file created with the following syntactically
   valid Heat::

     parameter_defaults:
       ExtraConfig:
         nova::compute::reserved_host_memory: 75000
         nova::cpu_allocation_ratio: 8.2

2. The compute node was deployed such that the commands below return
   something like the following::

    [root@overcloud-osd-compute-0 ~]# grep reserved_host_memory /etc/nova/nova.conf
    reserved_host_memory_mb=75000
    [root@overcloud-osd-compute-0 ~]# grep cpu_allocation_ratio /etc/nova/nova.conf
    cpu_allocation_ratio=8.2
    [root@overcloud-osd-compute-0 ~]#

Option 1 would put less load on the CI infrastructure and produce a
faster test but Option 2 tests the full scenario.

If a new derived parameter option is added, then the all-derivation-options
profile would need to be updated and the test would need to be updated
to verify that the new options were set.

Documentation Impact
====================

A new chapter would be added to the TripleO document on deploying with
derivation profiles.

References
==========

.. [1] `Deployment Plan Management specification <https://review.openstack.org/#/c/438918>`_
.. [2] `Spec for Ironic to retrieve NUMA node info <https://review.openstack.org/#/c/396147>`_
.. [3] `<https://github.com/Jaganathancse/Jagan/tree/master/mistral-workflow>`_
.. [4] `nova_mem_cpu_calc.py <https://github.com/RHsyseng/hci/blob/master/scripts/nova_mem_cpu_calc.py>`_
.. [5] `NUMA Topology in introspection data (ironic-python-agent) <https://review.openstack.org/#/c/424729/>`_
